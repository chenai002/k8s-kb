# GPU Operator 与 DRA（让 K8s 认识并精细分配 GPU）

> ⚠️ **待核实**：DRA（Dynamic Resource Allocation）处于从 alpha/beta 到 GA 的演进过程中（1.34 前后为关键节点），API 组（`resource.k8s.io`）与字段名可能变化。请以 K8s 官方文档与 NVIDIA GPU Operator 文档为准。

| 项 | 内容 |
| --- | --- |
| 方向 | C. 底座能力 |
| 难度 | ⭐⭐⭐ |
| 是什么 | GPU Operator = 自动装 GPU 全家桶；DRA = 精细分配资源的新机制 |
| 一句话 | 前者让 GPU「能用」，后者让 GPU「分得细」 |

---

## 一、先看一个场景

新买了一批 A100 节点，接进集群。你希望：

1. 驱动装好、GPU 能被 K8s 识别；
2. Pod 能申请 GPU：`nvidia.com/gpu: 1`；
3. 监控能看到 GPU 利用率、显存、温度；
4. 一个 8 卡的机器能拆给 8 个团队用；
5. 大模型需要 4 卡，要保证这 4 卡在**同一台机器、同一 NVLink 域**；
6. 小任务只需要 10 GB 显存，不要独占整卡。

前 3 条 → **GPU Operator**
第 4–6 条 → **传统扩展资源做不到，要 DRA**

---

## 二、先搞懂 GPU 在 K8s 里怎么表示

### 2.1 传统方式：扩展资源（Device Plugin）

K8s 内置支持 CPU/内存，其他资源靠 **Device Plugin** 注册：

```
nvidia-device-plugin（DaemonSet，每个 GPU 节点一个）
        │ 向 kubelet 注册
        ▼
kubelet 上报 Node 容量：
  nvidia.com/gpu: 8
        │
        ▼
Pod 申请：
  resources:
    limits:
      nvidia.com/gpu: "1"
```

**特点**：
- 只能申请**整数张**卡。
- `requests` 与 `limits` 必须相等（不可压缩资源）。
- **无法表达**「我要 40GB 显存」「我要同一 NVLink 的 4 张卡」「我要 MIG 的一个切片」。

### 2.2 MIG 是什么

**MIG（Multi-Instance GPU）** 是 A100/H100 等卡的能力：把一张物理卡切成多个独立的 GPU 实例（如 `1g.10gb`、`3g.40gb`）。

```
一张 A100 80GB
├── MIG 实例 1（1g.10gb）→ 给推理小任务
├── MIG 实例 2（1g.10gb）
├── MIG 实例 3（3g.40gb）→ 给微调任务
└── ...
```

**好处**：提高利用率。
**麻烦**：传统 Device Plugin 只能把 MIG 实例当成「一种资源」暴露（如 `nvidia.com/mig-1g.10gb: 7`），配置繁琐且不灵活。

---

## 三、GPU Operator 做了什么

### 3.1 它管理的组件

| 组件 | 作用 |
| --- | --- |
| **NVIDIA 驱动** | 宿主机内核模块 + 用户态库（可用预编译容器安装） |
| **Container Toolkit** | 让容器能访问 GPU（`nvidia-container-runtime`） |
| **Device Plugin** | 向 kubelet 注册 `nvidia.com/gpu` |
| **DCGM Exporter** | 暴露 GPU 指标（利用率、显存、温度、功耗、ECC） |
| **GPU Feature Discovery（GFD）** | 给节点打标签：型号、显存、MIG 能力、驱动版本 |
| **MIG Manager** | 自动配置 MIG 切片 |
| **Node Status Exporter** | 上报节点 GPU 健康 |
| **Validator** | 自检组件 |

**价值**：以前装 GPU 节点要手工装驱动、配 runtime、装插件，版本一乱就炸。GPU Operator 用 **operator 模式**把这些全部自动化，并保证版本匹配。

### 3.2 原理

```
你给节点打标签 nvidia.com/gpu.present=true
        │
        ▼
GPU Operator（控制器）
  · 创建 driver DaemonSet（若用容器化驱动）
  · 创建 container-toolkit DaemonSet
  · 创建 device-plugin DaemonSet
  · 创建 dcgm-exporter DaemonSet
  · 创建 GFD DaemonSet
        │
        ▼
节点自动拥有：
  · nvidia.com/gpu 资源
  · 一系列节点标签（型号/显存/MIG）
  · :9400/metrics 指标
```

### 3.3 GFD 打的标签举例

```
nvidia.com/gpu.product=NVIDIA-A100-SXM4-80GB
nvidia.com/gpu.memory=81920
nvidia.com/gpu.count=8
nvidia.com/mig.capable=true
nvidia.com/cuda.driver.major=550
```

**这些标签是调度与配额的基础**（Kueue 的 ResourceFlavor 就用它们区分卡型）。

---

## 四、DRA 做了什么

### 4.1 为什么需要 DRA

传统 Device Plugin 的三个限制：

| 限制 | 后果 |
| --- | --- |
| 只能整数分配 | 小任务独占整卡，浪费 |
| 无法表达约束 | 「必须是同 NVLink 的 4 卡」无法保证 |
| 无法延迟决策 | 资源类型在调度时固定，不能根据需求动态组合 |

### 4.2 DRA 的核心抽象

| 对象 | 类比 | 作用 |
| --- | --- | --- |
| **DeviceClass** | 「商品分类」 | 定义一类设备（如 `gpu.nvidia.com`）及其驱动 |
| **ResourceSlice** | 「库存清单」 | 由驱动上报，描述节点上有哪些设备及其属性 |
| **ResourceClaim** | 「采购单」 | Pod 声明「我要什么样的设备」 |
| **ResourceClaimTemplate** | 「采购单模板」 | 每个 Pod 生成一份独立 claim |

### 4.3 关键区别：从「计数」到「描述」

**传统**：
```yaml
resources:
  limits:
    nvidia.com/gpu: "4"     # 我要 4 张卡（不知道是哪 4 张）
```

**DRA**：
```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: four-gpus
spec:
  spec:
    devices:
      requests:
        - name: gpus
          exactly:
            deviceClassName: gpu.nvidia.com
            count: 4
            selectors:
              - cel:
                  expression: |
                    device.attributes["gpu.nvidia.com"].memory >= 80
                    && device.attributes["gpu.nvidia.com"].product == "A100"
---
apiVersion: v1
kind: Pod
spec:
  resourceClaims:
    - name: gpus
      resourceClaimTemplateName: four-gpus
  containers:
    - name: trainer
      resources:
        claims:
          - name: gpus
```

**原理**：
1. Pod 创建时，调度器看到 `ResourceClaim`。
2. 调度器根据 `selectors`（CEL 表达式）在 `ResourceSlice` 中查找满足条件的设备。
3. 找到后**在调度阶段就完成分配**，写回 claim。
4. kubelet 在该节点上准备设备（MIG 配置、驱动挂载等）。

**好处**：
- 可表达「显存 ≥ 40GB」「同一 NUMA」「同一 NVLink 域」等约束。
- 支持细粒度（MIG 切片）与动态组合。
- 分配在调度阶段完成，避免「调度完发现资源不匹配」。

### 4.4 DRA 处理「8 卡机器拆给 8 个人」

```
一张 A100 80GB
  → ResourceSlice 暴露 8 个 MIG 设备（或若干可组合切片）
  → 8 个不同的 ResourceClaim 各自分配一个 MIG 实例
  → 每个 Pod 只看得到自己的 MIG 实例
```

---

## 五、两者关系

```
┌───────────────────────────────────────────────┐
│ GPU Operator                                   │
│ · 装驱动 / runtime / 插件                       │
│ · 上报 GPU 指标、打节点标签                     │
│ · 配置 MIG                                      │
└───────────────────┬───────────────────────────┘
                    │ 让节点「有 GPU」
                    ▼
┌───────────────────────────────────────────────┐
│ DRA（K8s 核心机制）                             │
│ · 精细表达设备需求                              │
│ · 调度阶段分配                                  │
│ · 支持 MIG/拓扑/属性约束                        │
└───────────────────────────────────────────────┘
```

**GPU Operator 是「安装与运维」，DRA 是「分配机制」。** 两者不是替代关系。

> 补充：NVIDIA 也提供 DRA driver（`nvidia-dra-driver-gpu`），为 DRA 提供 GPU 设备清单与准备逻辑。⚠️ 名称与成熟度需核实。

---

## 六、怎么用（入门）

### 6.1 装 GPU Operator

```bash
# 用 Helm 安装（示意）
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm install gpu-operator nvidia/gpu-operator \
  -n gpu-operator --create-namespace \
  --set driver.enabled=true \
  --set toolkit.enabled=true

# 验证
kubectl get pods -n gpu-operator
kubectl describe node <gpu-node> | grep -i nvidia.com
```

### 6.2 申请 GPU（传统方式）

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: cuda
      image: nvcr.io/nvidia/cuda:12.4.0-base-ubuntu22.04
      command: ["nvidia-smi"]
      resources:
        limits:
          nvidia.com/gpu: "1"
```

**观察**：`nvidia-smi` 只看到一张卡（隔离效果）。

### 6.3 监控

```promql
# GPU 利用率
DCGM_FI_DEV_GPU_UTIL
# 显存使用
DCGM_FI_DEV_FB_USED
```

---

## 七、有什么坑

### 7.1 驱动版本

宿主机内核升级、驱动升级、CUDA 版本、容器镜像里的 CUDA 版本，四者要兼容。GPU Operator 能降低风险，但不能消除。

### 7.2 整数分配的浪费

一个只需要 10GB 显存的推理服务独占整张 80GB 卡。解决：MIG 或 DRA 细粒度。

### 7.3 多卡作业的拓扑

4 卡任务如果分到 2 台机器各 2 卡，性能会崩（没有 NVLink，走网络）。需要：
- 节点标签 + nodeAffinity；
- DRA 的拓扑约束；
- 或直接用整机节点跑大任务。

### 7.4 共享 GPU 的隔离问题

时间片共享（time-slicing）能让多个 Pod 共用一张卡，但**没有显存隔离**，一个 OOM 会拖垮别人。只适合开发/测试。

### 7.5 MIG 的灵活性代价

MIG 切片需要重新配置（可能中断正在跑的任务），且切片规格有限。

### 7.6 DRA 版本差异

不同 K8s 版本的 DRA API 有变化，升级要小心。**生产使用前确认是否 GA 并评估兼容性。**

---

## 八、小白容易混淆的点

| 混淆 | 澄清 |
| --- | --- |
| **GPU 是 K8s 内置资源吗？** | 不是，靠 Device Plugin 注册为扩展资源 |
| **GPU Operator 是 K8s 官方组件吗？** | 不是，是 NVIDIA 的 Operator |
| **DRA 是 NVIDIA 的吗？** | 不是，是 K8s 核心特性；NVIDIA 提供驱动实现 |
| **MIG 和 DRA 的关系？** | MIG 是硬件能力，DRA 是表达与分配它的机制 |
| **time-slicing 和 MIG 的区别？** | 时间片共享同卡不同时；MIG 是硬件切分，各自独立显存与算力 |
| **`nvidia.com/gpu` 能填 0.5 吗？** | 不能，扩展资源只能整数；要细粒度得用 MIG/DRA |
| **requests 和 limits 为什么必须相等？** | 因为 GPU 是不可压缩资源，不支持超卖 |

---

## 九、面试会怎么问

**Q1：GPU 在 K8s 里怎么被调度？**
Device Plugin 向 kubelet 注册扩展资源，kubelet 上报到 Node，调度器按 `nvidia.com/gpu` 的 requests 过滤节点。

**Q2：GPU Operator 管什么？**
驱动、container toolkit、device plugin、DCGM 监控、GFD 标签、MIG 管理，用 operator 模式保证版本一致与自动化。

**Q3：DRA 解决什么问题？**
从「计数」到「描述」：支持属性约束（显存/型号/拓扑）、细粒度分配（MIG）、调度阶段决策。

**Q4：如何保证 4 卡在同一 NVLink 域？**
用 DRA 的拓扑/属性选择器，或用节点池 + 整机调度 + gang scheduling。

**Q5：共享 GPU 有什么风险？**
时间片共享无显存隔离，易被邻居 OOM 影响；MIG 隔离较好但规格受限。

---

## 十、延伸阅读

- NVIDIA GPU Operator 文档 ⚠️
- Kubernetes DRA 官方文档 ⚠️
- 同目录：[kueue.md](./kueue.md)（配额与排队）
- 同目录：[kserve-vllm.md](./kserve-vllm.md)（推理负载）
