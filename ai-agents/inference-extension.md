# Gateway API Inference Extension（让请求找到「最合适」的模型实例）

> ⚠️ **待核实**：本文基于截止 2026 年中的公开知识整理，**无法联网核对**。该项目（曾用名 Gateway API Inference Extension / InferencePool）版本演进较快，API 版本、`InferencePool` 字段名、是否 GA 请以 `kubernetes-sigs/gateway-api-inference-extension` 官方仓库为准。

| 项 | 内容 |
| --- | --- |
| 方向 | A. 在 K8s 上跑 Agent 的「模型路由」 |
| 难度 | ⭐⭐⭐⭐ |
| 是什么 | Gateway API 的扩展，为「推理负载」提供智能路由 |
| 一句话 | 普通负载均衡分流量，它分「最划算的算力」 |

---

## 一、先看一个场景

你部署了 4 个 vLLM 实例跑同一个模型。用户请求来了，该发给谁？

普通的 `round_robin` 会让每个实例轮流接。

但推理场景有三个特殊性：

### 1. 每个人的请求「重量」不同

- 请求 A：问「今天天气怎样」（10 个 token，算 0.5 秒）
- 请求 B：让它总结一份 50 页的 PDF（50000 个 token，算几分钟）

如果轮询分配，B 落到哪个实例，那个实例就被占满，后面来的小请求都要排队。**这叫「队头阻塞」**。

### 2. 前缀缓存决定了「谁来算最便宜」

vLLM 会把相同前缀的 KV Cache 存在显存里（prefix caching，见 [kserve-vllm.md](./kserve-vllm.md)）。

如果 100 个请求都用同一个 5000 字的系统提示：
- 打到**同一个实例**：只用算一次前缀，后续命中缓存，飞快。
- 打散到 4 个实例：每个实例都要重新算一次前缀，白白浪费。

**所以：相同前缀的请求应该尽量落到同一实例。**

### 3. 每个实例的「空闲程度」不同

实例 1 正在跑 3 个长请求，实例 2 空着。轮询会让新请求有 50% 概率去排队。

**结论**：推理需要的是「**知道每个后端当前负载 + 缓存内容**的智能路由」，而不是简单的轮询。

这就是 Inference Extension 要解决的。

---

## 二、它做了什么

用一句话：**把「哪些推理实例可用、各自多忙、缓存了什么」暴露给网关，让网关做出更好的选择。**

具体提供：

1. **`InferencePool` 资源**：表示「一组推理实例」。
2. **Endpoint Picker（EPP）**：一个小服务，负责根据实时指标选出最佳实例。
3. **与 Gateway API 集成**：通过 `HTTPRoute` 把请求路由到 `InferencePool`。
4. **指标采集**：从每个推理实例收集队列长度、KV Cache 使用率、活跃请求数等。

---

## 三、架构：为什么要多一个组件

普通的 Gateway API 路由是这样的：

```
Gateway（Envoy）──根据 HTTPRoute──▶ Service ──kube-proxy──▶ Pod
```

问题：**网关不知道后端忙不忙**。Envoy 只能做「基于连接数」的猜测。

Inference Extension 的架构：

```
                        ┌──────────────────────────┐
  请求 ──▶ Gateway      │  Endpoint Picker (EPP)   │
        （Envoy）  ◀────│  · 从各实例拉指标          │
             │  ▲      │  · 计算最佳实例            │
             │  │      │  · 返回给它（ext_proc）    │
             ▼  │      └────────────┬─────────────┘
        ┌───────────────────────┐    │ 指标
        │  vLLM Pod 1  (busy)   │◀───┤
        │  vLLM Pod 2  (idle)   │◀───┤
        │  vLLM Pod 3  (cached) │◀───┤
        └───────────────────────┘    │
                                     └── 采集队列/缓存指标
```

**关键交换**：
- 网关不再自己选后端，而是**问 EPP**「这个请求该去哪」。
- EPP 通过 **ext_proc（External Processing）** 过滤器与 Envoy 交互：Envoy 把请求头/体发给 EPP，EPP 回复「发往哪个 endpoint」。
- EPP 的决策依据是它从各实例**主动拉取**的指标。

### 3.1 为什么放在 ext_proc 而不是改 Envoy 代码

- 不需要 fork Envoy，可独立升级。
- 决策逻辑用任何语言写（通常是 Go）。
- 仍复用 Envoy 的连接管理、TLS、观测。

代价：多一次 RPC 调用（通常 <1ms），以及 EPP 本身需要高可用。

---

## 四、实现原理（拆开看）

### 4.1 核心资源

```yaml
apiVersion: inference.networking.x-k8s.io/v1alpha2      # ⚠️ 版本以官方为准
kind: InferencePool
metadata:
  name: llama-pool
  namespace: ai
spec:
  selector:
    matchLabels: { app: vllm }
  targetPortNumber: 8000
  extensionRef:
    name: llama-epp                    # 关联的 Endpoint Picker
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: llama-route
  namespace: ai
spec:
  parentRefs: [{ name: ai-gateway }]
  rules:
    - matches:
        - path: { type: PathPrefix, value: "/v1/chat" }
      backendRefs:
        - group: inference.networking.x-k8s.io
          kind: InferencePool
          name: llama-pool
```

**注意**：`backendRefs` 指向的不是 `Service`，而是 `InferencePool` —— 这就是扩展点。

### 4.2 EPP 的决策算法（调度打分）

EPP 通常按多个维度打分，选最高分：

| 维度 | 含义 | 权重倾向 |
| --- | --- | --- |
| **队列长度** | 该实例排队的请求数 | 越少越好（防队头阻塞） |
| **KV Cache 使用率** | 显存里缓存占了多少 | 越低越好（还有空间） |
| **活跃请求数** | 正在处理的请求 | 越少越好 |
| **前缀匹配** | 请求前缀是否命中该实例缓存 | 命中则大幅加权 |
| **LoRA 适配器** | 该实例是否加载了请求要的微调模型 | 必须匹配 |

**打分公式（示例）**：

```
score = w1 × (1 - queueLen/maxQueue)
      + w2 × (1 - kvcacheUsage)
      + w3 × prefixCacheHit
      + w4 × loraMatch
```

这套思路源自 vLLM 的 **load-aware scheduling** 与 KV-aware routing（社区里也有类似 "llm-d" 的项目在做同样的事）。

### 4.3 前缀缓存感知是怎么做到的

```
1. EPP 收到请求，看到 prompt 的前 N 个 token
2. 计算前缀哈希（通常取前若干 token 的 hash）
3. 查它维护的「实例 → 已缓存前缀集合」映射
4. 命中的实例获得加分
5. 同时把这个前缀加入「即将发送」的记账，避免同一时刻全部打到同一实例
```

**难点**：
- 前缀缓存的真实状态在 vLLM 内部，需要通过指标/事件暴露。
- 缓存会淘汰（LRU），EPP 的视图会过时，只能是**尽力而为**。

### 4.4 一次请求的完整流程

```
① 请求到达 Gateway（Envoy），匹配 HTTPRoute → InferencePool
② Envoy 通过 ext_proc 把请求发给 EPP
③ EPP：
   - 解析请求体里的 model、prompt
   - 拉取/读取各实例的实时指标
   - 计算前缀哈希，查缓存感知
   - 打分，选出 endpoint
④ EPP 返回 header（如 `x-gateway-destination-endpoint: 10.0.0.5:8000`）
⑤ Envoy 把请求直发该 Pod IP
⑥ 响应返回；统计更新（token、延迟）
```

**注意第 ⑤ 步**：Envoy **绕过 Service/kube-proxy**，直连 Pod IP（和 Istio sidecar 的做法一样）。

### 4.5 与自动扩缩的配合

推理的扩缩信号与 Web 不同：

| 指标 | 说明 |
| --- | --- |
| 队列长度 | 排队久了就该扩 |
| KV Cache 使用率 | 快满了该扩 |
| GPU 利用率 | 传统指标 |
| 每秒 token | 吞吐指标 |

EPP 的指标可以直接喂给 HPA / KEDA（通过 Prometheus Adapter），形成闭环：

```
流量上升 → 队列变长 → HPA 扩容 → 新实例注册到 InferencePool
        → EPP 自动纳入调度
```

---

## 五、怎么用（概念流程）

```bash
# 1. 装 Gateway API CRD + 一个支持 ext_proc 的实现（Envoy Gateway 等）
# 2. 装 Inference Extension CRD 与 EPP

# 3. 部署 vLLM（带指标暴露）
kubectl apply -f vllm-deployment.yaml

# 4. 创建 InferencePool
kubectl apply -f inferencepool.yaml

# 5. 创建 HTTPRoute 指向 InferencePool
kubectl apply -f httproute.yaml

# 6. 验证
kubectl get inferencepool -n ai
curl -H "Host: llama.example.com" http://$GW/v1/chat/completions -d '...'
```

**观察点**：`kubectl get inferencepool` 能看到实例数量与状态。

---

## 六、有什么坑

### 6.1 版本与稳定性

项目演进快，CRD 版本可能变化。生产使用前要评估是否 GA。

### 6.2 EPP 成为新单点

所有请求都要问 EPP，它挂了怎么办？
- 多副本 + 健康检查；
- Envoy 侧配置「EPP 不可用时降级为普通 LB」。

### 6.3 指标滞后

EPP 拉取指标有间隔（秒级），高并发下会决策过时，可能同时把大量请求打到同一实例。
- 用「预留/预扣」机制（发出前先记账）；
- 缩短采集间隔。

### 6.4 与 Service 的兼容

绕过了 kube-proxy，意味着 **NetworkPolicy 仍作用于 Pod 级**，但 Service 级的一些特性（如 sessionAffinity）不生效。

### 6.5 多模型 / LoRA

如果不同实例加载不同的 LoRA 适配器，EPP 必须知道这些信息，否则会把请求发到没加载该适配器的实例。

### 6.6 成本

更智能的路由需要更多指标与计算，延迟略增。对超低延迟场景要权衡。

---

## 七、小白容易混淆的点

| 混淆 | 澄清 |
| --- | --- |
| **它和 Ingress 的关系？** | 它是 Gateway API 的扩展，属于 L7 路由层，不是 Ingress |
| **它和 KServe 的关系？** | 层次不同。KServe 管「模型服务怎么部署」，Inference Extension 管「请求发给哪个实例」。可以配合使用 |
| **它和 AI Gateway 的关系？** | AI Gateway 管鉴权/限额/缓存；Inference Extension 管实例选择。通常 AI Gateway 在前面，Inference Extension 在它后面 |
| **能替代 Service 吗？** | 不能，它是路由层的扩展，服务发现仍依赖 Service/EndpointSlice 的机制（InferencePool 内部会生成） |
| **必须用 GPU 吗？** | 不需要，但对推理负载收益最大 |
| **是「模型路由器」吗？** | 可以这么理解，但它只在 K8s Gateway API 体系内工作 |

---

## 八、面试会怎么问

**Q1：为什么推理负载不能用普通负载均衡？**
请求耗时差异巨大导致队头阻塞；前缀缓存有亲和收益；需要感知队列与 KV Cache 状态。

**Q2：InferencePool 是什么？**
一种表示「一组推理实例」的资源，替代 HTTPRoute 中 Service 作为 backendRef，让扩展组件能接管 endpoint 选择。

**Q3：EPP 怎么工作？**
作为 ext_proc 服务接收 Envoy 请求，依据从各实例采集的指标（队列、KV Cache、前缀命中）打分选实例，返回目标 endpoint。

**Q4：前缀缓存感知的原理与局限？**
按 prompt 前 N token 哈希，记录到实例的映射；局限是缓存状态过时、需要预扣、跨实例一致性难保证。

**Q5：怎么和 HPA 配合？**
暴露队列长度/KV Cache 等指标，通过 Prometheus Adapter 喂给 HPA/KEDA，实现按推理特性扩缩。

---

## 九、延伸阅读

- gateway-api-inference-extension 官方仓库 ⚠️
- vLLM 的 prefix caching / KV-aware routing 文档
- 同目录：[kserve-vllm.md](./kserve-vllm.md)
- 同目录：[ai-gateway.md](./ai-gateway.md)
- [../istio.md](../istio.md)（Envoy 与流量治理基础）
