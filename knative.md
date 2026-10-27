# Knative 入门：它做什么，以及它是怎么实现的

> 面向「听说过 Knative 但没真正用过」的读者。
> 读完你应该能回答：Knative 解决什么问题、它和 K8s Deployment 有什么不同、`Knative Serving` 和 `Eventing` 各管什么、冷启动到底发生在哪一步。

---

## 目录

- [一、先看一个场景](#一先看一个场景)
- [二、Knative 是什么](#二knative-是什么)
- [三、Knative Serving（重点）](#三knative-serving重点)
  - [3.1 它做了什么](#31-它做了什么)
  - [3.2 核心对象](#32-核心对象)
  - [3.3 实现原理：一条请求的旅程](#33-实现原理一条请求的旅程)
  - [3.4 缩容到 0 与冷启动](#34-缩容到-0-与冷启动)
- [四、Knative Eventing](#四knative-eventing)
- [五、与其他方案的对比](#五与其他方案的对比)
- [六、生产使用的坑](#六生产使用的坑)
- [七、面试会怎么问](#七面试会怎么问)
- [附：术语表](#附术语表)

---

## 一、先看一个场景

你们有一个「图片转缩略图」的服务：

- 白天业务高峰，每秒 200 个请求；
- 深夜基本没人用，但偶尔有零星请求；
- 你们用 Deployment 跑了 10 个副本，24 小时在线。

问题：

- 深夜 10 个副本空转，**白烧钱**；
- 想缩到 0 又做不到（Deployment 最少 0 副本就没法接请求了，除非自己写个「唤醒」机制）。

于是你想：**「能不能像 Serverless 那样，没请求时缩到 0，来请求时自动拉起？」**

K8s 本身做不到「从 0 拉起」——因为它需要一个**始终存在的东西**接住第一个请求，再决定拉起谁。这就是 **Knative Serving** 要解决的。

---

## 二、Knative 是什么

### 2.1 一句话

> **Knative 是在 Kubernetes 之上构建 Serverless 平台的一组组件**，包含两块：
> - **Knative Serving**：请求驱动的服务 —— 自动扩缩（含缩到 0）、流量切分、版本管理；
> - **Knative Eventing**：事件驱动的绑定 —— 把「事件源」和「消费者」解耦。

### 2.2 它不是什么

| 常见误解 | 事实 |
| --- | --- |
| 它是一个 FaaS（函数平台） | 它是**服务**平台，跑的是容器，不是函数 |
| 它替代 K8s | 它在 K8s 之上，用 CRD 扩展 |
| 它只能在云上用 | 任何 K8s 集群都能装 |
| 它只是个自动扩缩器 | 还有流量治理、版本管理、事件模型 |

### 2.3 历史与现状

- 2018 年由 Google 发起，后成为 CNCF 项目（已毕业 ⚠️ 状态需核实）。
- 一度想推出「Knative 通用 API」，后来回归 K8s 生态，Serving 用 `networking.k8s.io` Ingress 或 Gateway API。
- 组件上：早期有 `Serving`、`Eventing`、`Build` 三大块；**Build 已剥离**（被 Tekton 等替代）。

---

## 三、Knative Serving（重点）

### 3.1 它做了什么

把「一个长期在线、有一定副本数的服务」升级为「**按需启动、自动扩缩、可灰度**」的服务：

| 能力 | 说明 |
| --- | --- |
| **缩到 0**（scale to zero） | 没有请求时把副本降到 0，来请求再拉起 |
| **请求驱动的扩缩**（KPA） | 按并发请求数（不是 CPU）扩缩 |
| **流量切分** | 一个服务多个版本，按百分比或 header 分流 |
| **版本管理** | 每次修改生成新的 Revision，可回滚 |
| **按请求路由** | 请求先到「数据面」，再由它转发给具体 Revision |
| **配置与密钥挂载** | 简化 ConfigMap/Secret 的使用与自动重载 |

### 3.2 核心对象

```
Service（Knative Service，简称 ksvc）
  └── Configuration（配置：每次修改产生一个 Revision）
        └── Revision（一次不可变的快照，对应一组 Pod）
              └── Deployment / Pod

Route（路由规则）
  └── 指向一个或多个 Revision，带权重
```

**小白理解**：
- `Service`：用户看到的「服务名」，一个稳定的 URL。
- `Configuration`：当前的配置模板，改它就产生新版本。
- `Revision`：**不可变的版本快照**，一旦创建就不改，只能新建或删除。
- `Route`：把流量分给哪些 Revision、各占多少。

一个最小示例：

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: thumbnail
  namespace: media
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/min-scale: "0"
        autoscaling.knative.dev/max-scale: "50"
        autoscaling.knative.dev/target: "100"   # 每副本目标并发
    spec:
      containers:
        - image: registry.example.com/thumbnail:1.2.0
          ports:
            - containerPort: 8080
          resources:
            requests: { cpu: 100m, memory: 128Mi }
```

`kubectl get ksvc` 就能看到 URL，创建后即使没有 Pod，URL 也是可用的。

### 3.3 实现原理：一条请求的旅程

这是这篇文档最重要的部分。**Knative 能做到「0 副本还能接请求」，靠的是把「流量入口」和「实际运行」分离。**

#### 架构总览

```
                    ┌──────────────────────────────┐
   用户请求 ───────▶ │  入口层（Ingress / Gateway）   │
                    │  例如 Kourier / Istio /        │
                    │  Contour / Gateway API 实现    │
                    └──────────────┬───────────────┘
                                   │ 1. 查路由：这个 host 对应哪个 Revision？
                                   ▼
                    ┌──────────────────────────────┐
                    │  Activator（激活器）           │
                    │  · 没有就绪 Pod 时，先「接住」  │
                    │  · 请求触发扩容                │
                    │  · Pod 起来后转发过去           │
                    └──────────────┬───────────────┘
                                   │ 2. 请求扩容服务
                                   ▼
                    ┌──────────────────────────────┐
                    │ Autoscaler（KPA）             │
                    │  · 收集并发指标                │
                    │  · 决定副本数                  │
                    └──────────────┬───────────────┘
                                   │ 3. 修改 Deployment replicas
                                   ▼
                    ┌──────────────────────────────┐
                    │ Revision → Deployment → Pod   │
                    │  （实际运行用户容器）           │
                    └──────────────────────────────┘
```

#### 关键组件职责

| 组件 | 角色 | 做什么 |
| --- | --- | --- |
| **knative-serving 控制器**（`controller`） | 控制面 | 把 `Service` 翻译成 `Configuration`/`Route`/`Revision` |
| **Autoscaler** | 控制面 | 运行 KPA（Knative Pod Autoscaler），决定副本数 |
| **Activator** | 数据面（关键） | 没有就绪副本时，**先接住请求并在内存里排队**，触发扩容，等 Pod 就绪后转发 |
| **Queue-Proxy** | 数据面 | 每个 Pod 内的 sidecar，负责统计并发、排队、上报指标、处理优雅退出 |
| **Webhook** | 控制面 | 默认值填充、校验、sidecar 注入 |

> **注意 Queue-Proxy**：这是 Knative 的巧妙之处。每个业务 Pod 都带一个 `queue-proxy` sidecar，它才是真正被路由的入口，业务容器只在旁边。`queue-proxy` 统计「当前有多少并发请求」，并把数字报给 Autoscaler。

#### 完整时序（第一次冷请求）

```
① 用户请求 thumbnail.media.example.com
② 入口层（Ingress）匹配到 Route，但目标 Revision 当前 0 副本
③ 查找「有没有就绪后端」→ 没有
④ 把请求转发给 Activator（而不是直接黑洞）
⑤ Activator 先把连接「挂住」（请求在内存里等待，不返回错误）
⑥ Activator 通知 Autoscaler：「这个 Revision 有流量」
⑦ Autoscaler 修改 Deployment 的 replicas：0 → 1
⑧ kubelet 拉起 Pod（拉镜像、启动容器、启动 queue-proxy）
⑨ Pod 就绪（readiness 通过），queue-proxy 向 Activator 注册为可用后端
⑩ Activator 把挂住的请求转发给这个 Pod
⑪ 用户拿到响应（这就是「冷启动延迟」的来源）
```

**如果 1 秒内没起来怎么办？** Activator 会一直挂着（有超时保护），而不是立刻返回 503。这就是「缩到 0 还能用」的关键。

#### 稳定期（有副本时）

```
① 请求 → 入口层 → 直接路由到 Pod（queue-proxy）
② 不经过 Activator（省一跳，降低延迟）
③ queue-proxy 统计并发，定期上报
④ Autoscaler 根据并发数扩缩：
      并发/副本 > target → 扩容
      一段时间没请求 → 逐步缩（缩容窗口、稳定窗口）
⑤ 缩到 0 后，流量路径又回到「经过 Activator」
```

#### 「请求路径」是怎么被切换的

这是一个很重要的设计：**同一个 host 的流量路径会动态变化**：

| 状态 | 流量路径 |
| --- | --- |
| 有就绪副本 | Ingress → Pod（直连） |
| 0 副本 | Ingress → Activator →（扩容后）Pod |

实现上，Knative 会更新 Ingress/路由配置中的 backend，让入口层知道「现在该发给谁」。这也是为什么 Knative 需要和入口实现（Kourier/Istio/Contour/Envoy Gateway）配合。

### 3.4 缩容到 0 与冷启动

#### KPA 的算法（简化）

```
每副本目标并发 = target（默认 100）
当前并发 = sum(queue-proxy 上报的进行中请求)
期望副本 = ceil(当前并发 / target)
```

再加上：
- **稳定窗口**（stabilization window）：避免抖动，缩容时观察一段时间。
- **缩容延迟**（scale-to-zero-grace-period）：确认真的没流量才降到 0。
- **最小/最大副本数**：`min-scale`、`max-scale`。

#### 冷启动的几个阶段

| 阶段 | 耗时来源 | 优化手段 |
| --- | --- | --- |
| 扩容决策 | Autoscaler 周期 | 缩短上报间隔 |
| Pod 调度 | 节点资源是否够 | 节点池预留、`min-scale: 1` |
| 拉镜像 | 镜像大小、网络 | 预拉镜像、小镜像、镜像缓存 |
| 容器启动 | 应用初始化 | 缩短初始化、延迟加载 |
| 就绪探测 | probe 配置 | 合理设置 readiness |

**结论**：Knative 能帮你缩到 0，但**冷启动能不能接受，取决于应用本身**。如果服务要求 P99 < 200ms，基本只能 `min-scale >= 1`。

#### 预热与保底

| 配置 | 作用 |
| --- | --- |
| `min-scale: 1` | 始终保留 1 个副本，避免冷启动 |
| `min-scale: 0` + `target` 调小 | 更敏感地扩容 |
| **冻结/挂起**（部分实现支持） | 保留 Pod 内存、快速恢复 ⚠️ 需核实 |
| 预热池 | 自定义实现，用空请求预热 |

---

## 四、Knative Eventing

### 4.1 它做什么

Serving 解决「请求 → 服务」，Eventing 解决「**事件 → 服务**」：

- 事件源（Kafka、GitHub webhook、K8s 事件、对象存储）产生事件；
- 消费者（Service、Job、外部 HTTP）订阅事件；
- 中间做**过滤、路由、重试、死信**。

### 4.2 核心概念

| 对象 | 类比 | 说明 |
| --- | --- | --- |
| **Event Source** | 「水龙头」 | 产生事件（`PingSource`、`KafkaSource`、`ApiServerSource`、`SinkBinding`） |
| **Broker** | 「邮局」 | 事件中枢，接收并分发 |
| **Trigger** | 「订阅规则」 | 定义「什么事件 → 发给谁」，支持过滤 |
| **Sink** | 「收件人」 | 消费者地址（Knative Service、K8s Service、Job） |
| **Channel** | 「管道」 | 点对点的事件通道（Broker/Trigger 的替代） |

### 4.3 实现原理

```
事件源 ──▶ Broker ──▶（过滤）──▶ Trigger ──▶ Sink（Service）
              │
              └── 需要持久化时 → 底层通道实现（Kafka / InMemory / NATS）
```

- **CloudEvents** 是标准事件格式（`ce-*` header + JSON body），Eventing 全程用它在组件间传递。
- Broker 的实现通常是 **MTChannelBasedBroker** 或 **Kafka Broker**：配置一组通道，按 Trigger 分发。
- 投递语义：**至少一次（at-least-once）**，所以消费者必须幂等。
- 失败处理：重试策略 + 死信队列（`deadLetterSink`）。

### 4.4 一个例子

```yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: thumbnail-on-upload
  namespace: media
spec:
  broker: default
  filter:
    attributes:
      type: com.example.image.uploaded
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: thumbnail
```

**含义**：「当 `default` broker 上出现 `type=com.example.image.uploaded` 的事件时，调用 thumbnail 服务」。

此时 thumbnail 可以是 `min-scale: 0` 的——**事件来了自动拉起**。

---

## 五、与其他方案的对比

| 方案 | 定位 | 缩到 0 | 特点 |
| --- | --- | --- | --- |
| **Deployment + HPA** | 长期在线服务 | ❌ | 简单，但无法从 0 拉起 |
| **Knative Serving** | Serverless 服务 | ✅ | 请求驱动、版本管理、流量切分；需要入口层 |
| **KEDA** | 事件驱动扩缩 | 部分支持（0→1 依赖 HTTP Add-on） | 轻量，只做扩缩，不改服务模型 |
| **OpenFaaS** | FaaS | ✅ | 函数模型，生态独立 |
| **云厂商 Serverless 容器**（如 GCR/ACA/Fargate） | 托管 | ✅ | 免运维，但绑定云 |

**KEDA 与 Knative 的常见混淆**：

- KEDA 是 **扩缩器**，把指标（队列长度、Kafka lag）喂给 HPA；它对「HTTP 请求」的 0→1 支持需要额外的 HTTP Add-on，且不如 Knative 的 Activator 成熟。
- Knative 是 **完整服务模型**：Version、Route、Revision、流量切分、事件。
- 结论：**只想要「按队列长度扩缩」→ KEDA；想要「Serverless 体验」→ Knative。**

---

## 六、生产使用的坑

### 6.1 冷启动不可控

这是最大的坑。`min-scale: 0` 的 P99 可能是几秒。**在线服务建议 `min-scale >= 1`，或至少对关键路径预热。**

### 6.2 入口层依赖

Knative 必须有入口实现（Kourier / Istio / Contour / Gateway API）。这一层的能力（TLS、超时、限流）会直接影响行为，比如默认 60s 超时可能导致长请求被切断。

### 6.3 queue-proxy 的并发统计偏差

`target` 是按并发算的，但 queue-proxy 统计的是「进行中的请求数」，长请求会让它误判为高负载。
**对长耗时接口要调大 target 或改用 CPU 指标。**

### 6.4 缩容太激进

默认缩容较积极，流量抖动会导致频繁扩缩（Pod 反复创建，冷启动反复发生）。用 `scale-down-delay` 和稳定窗口缓解。

### 6.5 Revision 不可变带来的困惑

改一个环境变量就产生一个新 Revision，老 Revision 不会自动删（`revisionHistoryLimit` 控制）。大量实验会堆积很多 Revision / Deployment。

### 6.6 与 Service Mesh 的叠加

Knative + Istio 会有两层代理（Istio sidecar + queue-proxy），延迟与资源开销叠加，要评估是否必要。

### 6.7 可观测性

指标分散在 queue-proxy 与 autoscaler，需要专门看：
- `revision_request_concurrency`
- `autoscaler_desired_pods`
- `revision_request_count`

---

## 七、面试会怎么问

**Q1：Knative 是什么？**
K8s 之上的 Serverless 平台：Serving 提供请求驱动的自动扩缩（含缩到 0）与版本/流量管理；Eventing 提供事件驱动的发布订阅。

**Q2：怎么做到缩到 0 还能接请求？**
把入口与实际运行分离：没有就绪副本时，入口把请求交给 Activator，Activator 挂住请求并触发扩容，Pod 就绪后转发。

**Q3：KPA 和 HPA 的区别？**
KPA 按「并发请求数」扩缩，专为请求型负载设计，支持缩到 0；HPA 按资源/自定义指标，最小 1（除非特殊处理）。

**Q4：Revision 是什么？为什么不可变？**
一次配置快照，对应一组 Pod。不可变是为了支持精确的流量切分与回滚，避免「同一版本行为漂移」。

**Q5：冷启动怎么优化？**
小镜像、预拉、缩短初始化、`min-scale` 保底、合理探针、必要时预热。

**Q6：Knative 和 KEDA 的区别？**
KEDA 是扩缩器，Knative 是服务模型；KEDA 的 0→1 需要 HTTP Add-on，Knative 内建 Activator 机制。

---

## 附：术语表

| 术语 | 含义 |
| --- | --- |
| **ksvc** | Knative Service，一个稳定的服务入口 |
| **Revision** | 不可变的版本快照 |
| **Route** | 把流量分配给 Revision 的规则 |
| **Activator** | 0 副本时接住请求、触发扩容的组件 |
| **Queue-Proxy** | 每个 Pod 的 sidecar，统计并发与排队 |
| **KPA** | Knative Pod Autoscaler，按并发扩缩 |
| **Scale to Zero** | 无请求时副本降至 0 |
| **CloudEvents** | 标准化的事件格式 |
| **Broker / Trigger** | 事件中枢 / 订阅规则 |
| **Sink** | 消费者地址 |
| **min-scale / max-scale** | 副本上下限 |
| **target** | 每副本的目标并发数 |

> ⚠️ 本文基于截止 2026 年中的公开知识整理，当前无法联网核实。CNCF 毕业状态、具体组件名（如入口实现支持列表）、KPA 参数默认值请以官方文档为准。
