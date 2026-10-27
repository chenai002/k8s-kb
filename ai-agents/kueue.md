# Kueue（让大量 AI 任务排队领 GPU）

> ⚠️ **待核实**：Kueue 的 API 版本（`v1beta1`/`v1`）与字段细节可能变化，请以 `kubernetes-sigs/kueue` 官方文档为准。

| 项 | 内容 |
| --- | --- |
| 方向 | A / C. 资源调度 |
| 难度 | ⭐⭐ |
| 是什么 | Kubernetes 的「作业排队系统」 |
| 一句话 | 像银行取号：资源不够就排队，不许插队，不许占着不用 |

---

## 一、先看一个场景

你的团队有 8 张 GPU，但同时在跑：

- 一个 4 卡的微调任务（要跑 6 小时）
- 三个 1 卡的服务（长期）
- 一个刚提交的 4 卡训练任务

会发生什么？

```
训练任务提交 → 调度器发现只剩 1 张卡 → Pending
              → 卡在 Pending 里，永远等不到 4 张同时空出来
              → 因为别的任务会把零散的卡吃掉
```

**这就是「资源碎片」+「没有排队」的问题。**

而且更糟的是：**先到先得可能不公平**。研发部提交了 5 个任务把 GPU 占满，算法部的一个任务都跑不了。

**Kueue 要解决的就是**：
1. 排队（不是无脑 Pending）
2. 公平分配配额（部门之间）
3. 保证整任务原子性（要么全给，要么不给）
4. 提高利用率（借用闲置配额）

---

## 二、它做了什么

### 2.1 核心概念

| 概念 | 类比 | 说明 |
| --- | --- | --- |
| **ClusterQueue** | 银行的「窗口资源池」 | 集群级，定义总配额（如 8 卡、256Gi 内存） |
| **LocalQueue** | 「某个部门的取号机」 | 命名空间级，指向 ClusterQueue |
| **Workload** | 「排队号票」 | Kueue 为一个 Job 创建的内部对象 |
| **ResourceFlavor** | 「窗口类型」 | 区分 GPU 型号 / 节点池 / 云厂商 |
| **WorkloadPriorityClass** | 「VIP 通道」 | 高优先级可插队 |
| **Cohort** | 「窗口之间可以互相借」 | 多个 ClusterQueue 组成一个组，闲置配额可借用 |

### 2.2 它不管什么

| 不做 | 谁做 |
| --- | --- |
| 真正把 Pod 放到节点 | kube-scheduler |
| 扩容节点 | Karpenter / Cluster Autoscaler |
| 按指标扩缩 | HPA / KEDA |
| 定义作业 | Job / JobSet / MPIJob / RayJob |

**Kueue 是「进场闸门」**：先决定「这个作业现在能不能开始」，能开始才放行给 kube-scheduler。

---

## 三、实现原理（拆开看）

### 3.1 核心思路：准入（Admission）而非调度

```
用户提交 Job
    │
    ▼
┌─────────────────────────────┐
│ Kueue 拦截（webhook / 控制器）│
│  · 创建 Workload 对象         │
│  · 把 Job 挂起（suspend）     │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│ 队列排序                     │
│  · 优先级                     │
│  · 提交时间（FCFS）           │
└──────────────┬──────────────┘
               ▼
┌─────────────────────────────┐
│ 配额检查（能否整批满足？）     │
│  · 够 → 准入：取消 suspend    │
│  · 不够 → 继续排队            │
└──────────────┬──────────────┘
               ▼
        kube-scheduler 正常调度
```

**关键技巧**：Kueue 用 **`suspend: true`** 把 Job 挂起。被挂起的 Job **不会创建 Pod**，因此不占用任何真实资源，也不会被调度器反复尝试。这比「让 Pod Pending」优雅得多：

| 方式 | 缺点 |
| --- | --- |
| Pod Pending | 占满调度队列与 etcd 对象，事件刷屏，扩容器可能误扩容 |
| Job 挂起 | 几乎没有开销，等到有资源再真正创建 Pod |

### 3.2 配额与「借还」

```
Cohort（组）
├── ClusterQueue：team-a（配额 4 GPU）
├── ClusterQueue：team-b（配额 4 GPU）
└── 总 GPU：8

情况 1：team-a 只用 1 卡，team-b 想用 7 卡
        → team-b 可以借用 team-a 的闲置 3 卡（借）
        → 但 team-a 一旦需要，会「收回」（抢占/驱逐 team-b 的借用任务）
```

这就是 **borrowing + preemption**，它让集群利用率接近 100%，同时保证配额承诺。

### 3.3 与调度器的分工（重要区分）

```
Kueue        ：这个作业「能不能开始」（准入）
kube-scheduler：这些 Pod「放在哪台机器上」（绑定）
```

为什么要分两层？
- 配额是**逻辑**概念（部门/项目），与节点无关。
- 调度是**物理**概念（GPU 型号、拓扑）。

分开后，配额可以跨节点池统一管理。

### 3.4 多 Pod 作业的原子性（gang）

Distributed 训练需要 8 个 Pod 同时起来（否则互相等待死锁）。
Kueue 的准入是**整作业级别**的：要么 8 个都放行，要么都不放行。

> 这与「gang scheduling」（Volcano 的强项）思路一致；Kueue 通过准入保证不同时缺资源，Volcano 通过调度器保证同时绑定。

### 3.5 ResourceFlavor：区分 GPU 型号

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata:
  name: a100
spec:
  nodeLabels:
    nvidia.com/gpu.product: NVIDIA-A100-SXM4-80GB
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata:
  name: t4
spec:
  nodeLabels:
    nvidia.com/gpu.product: Tesla-T4
```

ClusterQueue 里可以给不同 flavor 分配不同配额：

```yaml
resourceGroups:
  - coveredResources: ["cpu", "memory", "nvidia.com/gpu"]
    flavors:
      - name: a100
        resources:
          - { name: "nvidia.com/gpu", nominalQuota: 8 }
      - name: t4
        resources:
          - { name: "nvidia.com/gpu", nominalQuota: 16 }
```

---

## 四、配置示例

### 4.1 ClusterQueue

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: team-a-cq
spec:
  namespaceSelector: {}                 # 允许哪些 ns 使用
  cohort: shared-gpu                    # 加入组，可互相借
  resourceGroups:
    - coveredResources: ["cpu", "memory", "nvidia.com/gpu"]
      flavors:
        - name: a100
          resources:
            - { name: "cpu", nominalQuota: "64" }
            - { name: "memory", nominalQuota: "256Gi" }
            - { name: "nvidia.com/gpu", nominalQuota: "4" }
  preemption:
    withinClusterQueue: LowerPriority
    reclaimWithinCohort: Any
```

### 4.2 LocalQueue

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  name: team-a-lq
  namespace: team-a
spec:
  clusterQueue: team-a-cq
```

### 4.3 提交作业

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: finetune-llama
  namespace: team-a
  labels:
    kueue.x-k8s.io/queue-name: team-a-lq    # ← 关键：告诉 Kueue 用哪个队列
spec:
  parallelism: 8                           # 需要 8 个 Pod
  completions: 8
  template:
    spec:
      containers:
        - name: trainer
          image: my-trainer:latest
          resources:
            limits:
              nvidia.com/gpu: "1"          # 8 Pod × 1 卡 = 8 卡（超出配额 4）
```

**预期**：作业创建后**没有 Pod**，`kubectl get workload` 能看到它排队中（`QuotaReserved: false`）。
直到配额够 8 卡（或借用成功），才会放行。

### 4.4 加优先级

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: WorkloadPriorityClass
metadata:
  name: urgent
value: 1000
---
# 在 Job 上：
# metadata.labels: kueue.x-k8s.io/priority-class: urgent
```

---

## 五、和 HPA / Cluster Autoscaler 的关系

| 组件 | 职责 | 与 Kueue 的交互 |
| --- | --- | --- |
| Kueue | 准入（能不能开始） | 决定是否创建 Pod |
| Cluster Autoscaler / Karpenter | 加节点 | 被挂起的作业不会触发扩容（因为没 Pod）→ **这是重要特性**，避免无意义的扩容 |
| HPA / KEDA | 按流量扩副本 | 与 Kueue 通常不用于同一类负载（HPA 用于服务，Kueue 用于批任务） |

**面试加分点**：Kueue 让「等待资源」不再表现为 Pending Pod，因此**集群扩容器不会被排队任务误触发**，避免了「为排队任务疯狂加机器」的浪费。

---

## 六、有什么坑

### 6.1 配额没配好 = 死锁

如果 ClusterQueue 的配额小于一个任务的需求，任务永远等不到。要检查 `nominalQuota ≥ 单任务需求`。

### 6.2 借用导致的惊群

大量 ClusterQueue 互相借用时，收回会触发驱逐。要给关键任务更高优先级，并设置 `reclaimWithinCohort` 策略。

### 6.3 与 Pod 直接创建不兼容

Kueue 主要针对 Job 类工作负载。直接创建的 Pod（裸 Pod、Deployment）**默认不受 Kueue 管理**（需要额外的策略或通过 Job 包装）。

### 6.4 拓扑与多卡

Kueue 管「配额」，但**不保证卡在同一台机器**。需要拓扑感知时，要配合 DRA 或 Volcano 的 gang scheduling。

### 6.5 指标与观测

`kubectl get workloads` 与 Kueue 的 Prometheus 指标要接上，否则「为什么我的任务不跑」很难排查。

---

## 七、小白容易混淆的点

| 混淆 | 澄清 |
| --- | --- |
| **Kueue 是调度器吗？** | 不是，它是**准入控制器**；真正的绑定由 kube-scheduler 完成 |
| **它是 Volcano 的替代品吗？** | 不完全。Kueue 强在配额与排队，Volcano 强在 gang 调度与队列策略；两者也可以一起用 |
| **和 YARN 的队列像吗？** | 思路很像（队列 + 配额 + 优先级），但 Kueue 是 K8s 原生 CRD 实现 |
| **它能让 GPU 利用率 100% 吗？** | 能提高（借用 + 排队减少碎片），但不能解决「模型本身只用 30% 显存」这类问题 |
| **为什么 Job 创建后没有 Pod？** | 因为被 Kueue suspend 了，这是正常现象 |

---

## 八、面试会怎么问

**Q1：Kueue 解决什么问题？**
多租户场景下的配额管理与排队，避免资源碎片与不公平分配；用 suspend 机制让等待不产生 Pending Pod。

**Q2：它和 kube-scheduler 的区别？**
Kueue 决定「作业能否开始」（配额/准入），scheduler 决定「Pod 放在哪个节点」。

**Q3：借用与抢占怎么工作？**
Cohort 内的 ClusterQueue 可借用闲置配额；配额主人需要时，Kueue 驱逐借用方的作业（按优先级）。

**Q4：为什么不直接让 Pod Pending 等？**
Pending Pod 占用控制面资源、触发误扩容、且无法表达「整批一起开始」的语义。

**Q5：分布式训练怎么保证同时起？**
Kueue 的准入是作业级的（要么全部放行，要么等待）；必要时配合 gang scheduling。

---

## 九、延伸阅读

- kubernetes-sigs/kueue 官方文档 ⚠️
- 同目录：[gpu-operator-dra.md](./gpu-operator-dra.md)（GPU 资源怎么表示）
- 同目录：[kserve-vllm.md](./kserve-vllm.md)（推理负载）
