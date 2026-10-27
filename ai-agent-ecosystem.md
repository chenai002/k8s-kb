# Kubernetes 社区与 AI Agent 相关的组件全景

> 面向「想在 K8s 上跑 Agent / 用 Agent 管 K8s」的从业者。
> 说明：本机当前无法联网，文档依据的是我掌握的知识（截止 2026 年中的公开信息）。**请把带 ⚠️ 标记的版本/状态当作"需复核"**，项目演进很快，落地方案前务必以官方仓库为准。
> 相关文档：[`envoy.md`](./envoy.md)、[`istio.md`](./istio.md)、[`FAQ.md`](./FAQ.md)

---

## 目录

- [一、先分清两个方向](#一先分清两个方向)
- [二、方向 A：在 K8s 上运行 AI Agent](#二方向-a在-k8s-上运行-ai-agent)
  - [2.1 Agent 的运行时形态与 K8s 的错配](#21-agent-的运行时形态与-k8s-的错配)
  - [2.2 Agent Sandbox（kubernetes-sigs）](#22-agent-sandboxkubernetes-sigs)
  - [2.3 网关与模型接入](#23-网关与模型接入)
  - [2.4 推理与模型部署](#24-推理与模型部署)
  - [2.5 MCP 服务端在 K8s 里的形态](#25-mcp-服务端在-k8s-里的形态)
  - [2.6 Workflow 与编排](#26-workflow-与编排)
- [三、方向 B：用 Agent 运维 K8s](#三方向-b用-agent-运维-k8s)
  - [3.1 MCP Server 类](#31-mcp-server-类)
  - [3.2 Agent Operator 类](#32-agent-operator-类)
  - [3.3 Agent 定义标准：kagent / agentic AI 规范](#33-agent-定义标准kagent--agentic-ai-规范)
- [四、方向 C：平台侧共性能力](#四方向-c平台侧共性能力)
- [五、一张全景图](#五一张全景图)
- [六、选型建议](#六选型建议)
- [七、面试与交流常见问题](#七面试与交流常见问题)
- [附：术语表](#附术语表)

> 📚 **每个项目的单独详解**：[`ai-agents/README.md`](./ai-agents/README.md)（面向小白的入门系列）
> ⚡ **一页纸速览**：[`ai-agents-cheatsheet.md`](./ai-agents-cheatsheet.md)（分层图 + 速查卡 + 口诀）

---

## 一、先分清两个方向

社区里说「K8s + AI Agent」，实际是两件不同的事，混在一起聊会很乱：

| 方向 | 问题 | 典型组件 |
| --- | --- | --- |
| **A. 在 K8s 上跑 Agent** | Agent 是无状态长活进程 + 需要安全沙箱执行代码，K8s 原生没有对应抽象 | Agent Sandbox、KServe、Gateway API Inference Extension、InferencePool |
| **B. 用 Agent 运维 K8s** | 让 LLM 能安全地读/写集群 | MCP Server for Kubernetes、kagent、Kubernetes MCP、Kubectl AI 类插件 |
| **C. 平台共性** | GPU 调度、模型缓存、可观测、成本 | NVIDIA GPU Operator、DRA、Kueue、Prometheus/Grafana AI 面板 |

下面分别展开。

---

## 二、方向 A：在 K8s 上运行 AI Agent

### 2.1 Agent 的运行时形态与 K8s 的错配

一个典型的 Agent 有这些特征：

| 特征 | 与 K8s 原生抽象的冲突 |
| --- | --- |
| 长会话、有状态（上下文、记忆） | Deployment 是无状态的，Pod 重建即丢 |
| 频繁创建/销毁的「执行环境」（跑工具、跑代码） | 用 Pod 起一个容器动辄几秒，且资源申请笨重 |
| 需要强隔离（执行不可信代码） | 普通容器共享内核，逃逸风险 |
| 突发性极强（一次任务可能起 100 个沙箱） | HPA 跟不上，节点扩容慢 |
| 需要访问外部工具/数据源 | 需要认证、网络策略、审计 |

社区因此产生了几个专门的抽象。

### 2.2 Agent Sandbox（kubernetes-sigs）⚠️

**是什么**：Kubernetes SIG 下的一个项目，目标是为「Agent 执行代码」提供**快、隔离、可批量**的沙箱抽象。

**核心思路**（基于公开设计讨论）：
- 引入一个新的 CRD（类似 `Sandbox` 或 `SandboxTemplate`），声明「我要一个隔离的执行环境」。
- 底层用 **gVisor / Kata Containers** 这类强隔离运行时，而不是普通 runc 容器。
- 通过**预热池（warm pool / snapshot）** 减少冷启动：预先准备好一批沙箱实例，Agent 需要时直接拿，实现亚秒级启动。
- 与 `RuntimeClass` 结合，让调度器知道哪些节点能跑沙箱。

**为什么不用 Pod**：
- Pod 的创建涉及调度、CNI、CSI、kubelet 一整套流程，冷启动常在 1–5 秒；
- Agent 场景一次任务可能要几十上百个短命沙箱，Pod 数量带来的控制面压力大；
- Pod 资源的回收（Terminating、finalizer）链路长。

**现状**：处于早期（alpha/设计阶段）。**面试时如果被问「K8s 原生支持 Agent 吗」，正确答案是：没有，社区在推进 Sandbox 这类抽象，当前生产上多是用 RuntimeClass + gVisor/Kata 自己搭。**

### 2.3 网关与模型接入

Agent 与模型之间的调用链路需要治理，这部分直接复用了 K8s 网关生态：

| 组件 | 作用 | 备注 |
| --- | --- | --- |
| **Gateway API** | L7 路由标准，Agent 网关的基础 | `HTTPRoute`/`GRPCRoute` 足够 |
| **Envoy AI Gateway** ⚠️ | 面向 LLM 的网关：按模型路由、token 限流、Provider 抽象、语义缓存 | 基于 Envoy + Gateway API |
| **Istio / Envoy** | mTLS、可观测、限流 | 见 [`istio.md`](./istio.md) |
| **LiteLLM Proxy**（非 K8s 原生但常用） | 多 Provider 统一 OpenAI 兼容接口、配额、成本统计 | 通常以 Deployment 部署 |

**Agent 网关要解决的问题**：
- 多模型供应商路由（OpenAI / Anthropic / 自建 vLLM）。
- **Token 级限流**（不是 QPS）：按输入/输出 token 计费与配额。
- 密钥管理（Agent 不持有上游 API Key）。
- 语义缓存（相同/相似 prompt 复用结果，省成本）。
- 审计与成本归因（哪个 Agent、哪个 namespace 花了多少）。

### 2.4 推理与模型部署

Agent 背后通常是自建模型服务：

| 组件 | 定位 |
| --- | --- |
| **KServe** | CNCF 模型服务框架，支持 vLLM/Triton/HuggingFace runtime，支持 autoscaling（含缩到 0） |
| **Gateway API Inference Extension** ⚠️ | SIG Network 推动，为推理负载提供 **InferencePool** + 前缀缓存感知路由（prefix-cache aware） |
| **KubeRay** | Ray 集群 Operator，用于分布式推理与训练 |
| **NVIDIA GPU Operator** | 驱动、device plugin、MIG、DCGM 监控 |
| **Dynamic Resource Allocation（DRA）** | K8s 1.34 起 GA 方向，GPU 细粒度分配（MIG 切片、拓扑感知） |
| **Kueue** | 批调度与配额队列，AI 训练/推理任务排队 |
| **Volcano / Koordinator** | 大模型训练与推理的 gang 调度、NUMA/GPU 拓扑 |

**面试要点**：推理场景的调度需求与 Web 服务不同 —— 需要 **GPU 拓扑感知、前缀缓存亲和、缩到 0、长请求的优雅扩缩**，这些都在推动 K8s 调度与网关能力演进。

### 2.5 MCP 服务端在 K8s 里的形态

MCP（Model Context Protocol）让 Agent 访问外部工具。在 K8s 里部署 MCP Server 的几种模式：

| 模式 | 说明 | 优缺点 |
| --- | --- | --- |
| **每 Agent 一个 sidecar** | MCP Server 与 Agent 同 Pod | 隔离好、访问本地资源方便；资源浪费 |
| **共享服务（Deployment）** | 多个 Agent 调用同一个 MCP Service | 资源高效；需做鉴权与多租户 |
| **按需拉起（on-demand）** | 调用时创建 Pod/沙箱，用完销毁 | 隔离强；冷启动问题 |
| **网关聚合（MCP Gateway）** ⚠️ | 一个网关聚合多个 MCP Server，统一鉴权与路由 | 便于治理；网关成为热点 |

社区相关的非官方/半官方组件：
- **`mcp-server-kubernetes`**（Flux159 等）—— 第三方，给 LLM 操作集群。
- **`kubernetes-mcp-server`**（manusa）—— 第三方，支持多集群、多工具。
- **Docker MCP Gateway / ToolHive** ⚠️ —— 面向 MCP Server 的部署与网关（非 K8s SIG 官方）。

> 注意：「MCP Server for Kubernetes」目前主要是**社区/第三方**实现，Kubernetes 官方 SIG 并没有维护一个"MCP 网关"项目。面试时不要把它们说成官方组件。

### 2.6 Workflow 与编排

多 Agent 协作、长时间任务需要工作流引擎：

| 组件 | 说明 |
| --- | --- |
| **Argo Workflows** | CNCF 毕业项目，DAG 编排，适合把 Agent 步骤当 Task |
| **Kueue** | 队列与配额，适合大量短任务排队 |
| **Tekton** | 更偏 CI，但可用于 Agent 流水线 |
| **LangGraph / Temporal**（应用侧） | 状态机与持久化执行，通常作为 Agent 应用的一部分跑在 K8s 上 |

---

## 三、方向 B：用 Agent 运维 K8s

这一块社区更活跃，也更「K8s 原生」。

### 3.1 MCP Server 类

| 项目 | 性质 | 能力 |
| --- | --- | --- |
| **kubernetes-mcp-server**（manusa） | 社区（非 SIG） | 多集群、pods/logs/events CRUD、Helm、kubectl 等价操作；支持 stdio 与 HTTP 模式 |
| **mcp-server-kubernetes**（Flux159） | 社区 | kubectl/Helm 常用操作，偏轻量 |
| **kubectl-mcp-server**（rohitg00） | 社区 | 面向 Agent 的 kubectl 封装 |
| **Kubernetes MCP（各云厂商）** | 云厂商 | 托管集群的官方 MCP（如各类 Serverless 平台的 MCP Server） |

**共同问题，也是面试重点**：
- **权限**：MCP Server 用哪个 ServiceAccount？是否只读？
- **越权风险**：LLM 可能被 prompt injection 诱导执行 `kubectl delete`。
- **审计**：所有操作要落到审计日志，能追溯到「哪个 Agent/对话」。
- **速率与爆炸半径**：限制可操作 namespace，禁止删除 CRD/namespace/PV。

### 3.2 Agent Operator 类

| 项目 | 性质 | 说明 |
| --- | --- | --- |
| **kagent** | 开源（CNCF Sandbox 方向 ⚠️） | 用 CRD 定义 Agent（`Agent`、`Tool`、`ModelConfig`），在 K8s 里运行 Agent Runtime，内置 K8s 工具集、多 Agent 协作、A2A 协议支持 |
| **Google Cloud "kube-agents"** ⚠️ | Google 开源 | 面向 K8s 的 Agent 框架原型，与 kagent 思路相近 |
| **Kubernetes AI Toolchain Operator（KAITO）** | SIG Cloud Provider / Azure 推动 | 自动在 K8s 上部署模型推理（GPU 资源自动配置），偏基础设施 |
| **Kubeflow / KServe 的 Agent 相关扩展** | 生态 | 偏模型服务，不是 Agent 编排 |

**kagent 的核心抽象**（值得记住，是「Agent 即 K8s 资源」的代表）：
- `Agent`：声明式定义「用哪个模型 + 哪个系统提示 + 可调用哪些工具」。
- `ModelConfig`：模型端点与密钥（可对接 OpenAI/Ollama/vLLM）。
- `Tool`/`ToolServer`：工具注册（K8s 查询、Prometheus 查询、自定义 HTTP API）。
- 运行形态：每个 Agent 是一个 Pod（自带 web UI / API），通过工具与集群交互。
- 协议：支持 **A2A（Agent-to-Agent）**，让多个 Agent 互相调用。

### 3.3 Agent 定义标准：kagent / agentic AI 规范

社区在讨论「K8s 里的 Agent 应该长什么样」：

| 讨论中的抽象 | 含义 |
| --- | --- |
| **Agent CRD** | 把「Agent 的提示词、模型、工具、权限」变成声明式资源 |
| **Tool CRD** | 工具（函数）的声明式注册，带 schema 与权限 |
| **Agent 生命周期** | 启动、暂停、恢复、销毁；长会话的持久化 |
| **A2A 协议** | Agent 之间的标准通信（Google 推动，逐渐成为事实标准 ⚠️） |
| **策略与护栏** | 用 Kyverno/Gatekeeper/OPA 约束 Agent 能创建的资源类型 |

---

## 四、方向 C：平台侧共性能力

无论哪个方向，都绕不开这些「K8s 原生 + AI 加速」的组件：

| 能力 | 组件 | 说明 |
| --- | --- | --- |
| GPU 管理 | NVIDIA GPU Operator、AMD ROCm | 驱动、device plugin、MIG、监控 |
| 细粒度资源 | **DRA**（Dynamic Resource Allocation） | 1.34 起核心方向，支持 MIG 切片、拓扑感知、按需申请 |
| 批调度 | Kueue、Volcano | 队列、配额、gang scheduling |
| 弹性 | KEDA、Karpenter | KEDA 按队列/事件扩缩，Karpenter 快速扩节点 |
| 安全沙箱 | gVisor（`runsc`）、Kata Containers、MicroVM（Firecracker） | Agent 执行不可信代码的隔离层 |
| 网络 | Cilium（L7 策略）、Istio | Agent 访问工具/模型的服务间治理 |
| 可观测 | Prometheus、OTel、Langfuse（应用侧） | token 消耗、延迟、工具调用成功率 |
| 策略 | Kyverno、Gatekeeper | 限制 Agent 能创建的资源 |
| GitOps | Argo CD、Flux | Agent 产生的变更走 PR 而非直接 apply |

---

## 五、一张全景图

```
                         ┌─────────────────────────────────────┐
                         │         用户 / 业务系统              │
                         └───────────────┬─────────────────────┘
                                         │
                         ┌───────────────▼─────────────────────┐
                         │  Agent Gateway（Gateway API +       │
                         │  Envoy / Envoy AI Gateway ⚠️）       │
                         │  · 鉴权  · token 限流  · 语义缓存     │
                         └───────┬─────────────────┬───────────┘
                                 │                 │
             ┌───────────────────▼───┐   ┌─────────▼──────────────┐
             │  Agent Runtime        │   │  模型服务                │
             │  (kagent 等 CRD/Pod)  │──▶│  vLLM / KServe / Triton  │
             │  · 提示词/记忆         │   │  · InferencePool 路由 ⚠️ │
             │  · A2A 协作           │   └────────────────────────┘
             └───────┬───────────────┘
                     │ 调用工具
        ┌────────────┼─────────────────────────────┐
        ▼            ▼                             ▼
┌───────────────┐ ┌──────────────────┐   ┌────────────────────────┐
│ MCP Server    │ │ Agent Sandbox    │   │ K8s API（读写）         │
│ （工具接入）   │ │ gVisor/Kata ⚠️    │   │ 经 MCP + 最小权限 SA     │
└───────────────┘ └──────────────────┘   └────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│ 平台底座：K8s + GPU Operator + DRA + Kueue + Karpenter + Cilium  │
│          + Kyverno（护栏）+ Prometheus/OTel（可观测）+ Argo CD    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 六、选型建议

### 6.1 按「你要解决什么问题」选

| 你的需求 | 推荐起点 | 理由 |
| --- | --- | --- |
| 让运维同事用自然语言查集群 | kubernetes-mcp-server + 只读 SA + 聊天客户端 | 最轻，风险可控 |
| 让 Agent 自动处理告警/巡检 | kagent（或自建 Agent + MCP） | CRD 化、可审计 |
| Agent 要执行不可信代码 | gVisor/Kata + RuntimeClass；关注 Agent Sandbox | 隔离优先 |
| 自建模型推理 | KServe + vLLM + GPU Operator | 成熟度最高 |
| 多模型统一接入与配额 | 网关层（Envoy AI Gateway / LiteLLM） | 避免每个 Agent 自带 Key |
| 大规模短任务沙箱 | Kueue + 快启动运行时 + 预热池 | 控制面与冷启动 |

### 6.2 落地时的五个红线

1. **权限**：Agent 的 ServiceAccount 默认只读；写操作必须走审批或 GitOps。
2. **隔离**：执行代码的沙箱用 gVisor/Kata，不共享内核；网络默认拒绝。
3. **可审计**：每次工具调用落审计日志，能追溯到对话与用户。
4. **成本**：token 与 GPU 都要有配额与看板，否则 Agent 会烧钱。
5. **回滚**：Agent 产生的变更走 PR，不直接 `kubectl apply` 到生产。

---

## 七、面试与交流常见问题

**Q1：K8s 原生支持 AI Agent 吗？**
不支持。K8s 的抽象是「无状态服务 + 有状态集」，Agent 需要的是「长会话 + 快速隔离沙箱 + 工具调用治理」，社区在通过 Agent Sandbox、RuntimeClass、网关等组合来实现，尚无官方一等公民抽象。

**Q2：为什么用 gVisor/Kata 而不是普通容器跑 Agent 代码？**
Agent 会生成并执行代码，属于不可信输入。普通容器共享内核，逃逸即控节点；gVisor 用用户态内核拦截 syscall，Kata 用轻量 VM，隔离强得多。代价是启动慢、系统调用兼容性差。

**Q3：MCP 是什么？和 K8s 什么关系？**
MCP 是 Agent 与工具之间的协议。K8s 里没有官方 MCP 组件，但 MCP Server 很适合以 Deployment/sidecar 形式部署，用 Service 暴露、用 RBAC/NetworkPolicy 做治理。

**Q4：用 Agent 运维集群最大的风险？**
prompt injection + 过宽权限。防御是：最小权限 SA、只允许特定资源/namespace、写操作走审批、全套审计。

**Q5：Agent 的冷启动优化思路？**
预热池、快照恢复（VM/Firecracker）、简化 CNI/CSI 依赖、把 Agent Runtime 常驻而只销毁沙箱。

**Q6：推理负载和 Web 负载调度的区别？**
GPU 拓扑、显存与批处理、prefix cache 亲和（同一前缀的请求要打到同一实例）、缩到 0、长请求的优雅扩缩容。

**Q7：kagent 属于什么？**
一个开源项目，用 CRD 把 Agent/ModelConfig/Tool 声明式化，在 K8s 中运行 Agent，支持 A2A。不是 Kubernetes 官方 SIG 项目。

**Q8：如果被问到「社区有哪些组件」，怎么答不跑偏？**
分三层：**运行 Agent**（Sandbox、KServe、推理网关）、**用 Agent 操作 K8s**（MCP Server、kagent）、**平台底座**（GPU Operator、DRA、Kueue、gVisor）。并强调「官方 vs 社区/第三方」的区别。

---

## 附：术语表

| 术语 | 含义 |
| --- | --- |
| **Agent** | 由 LLM 驱动、能规划与调用工具完成任务的程序 |
| **MCP** | Model Context Protocol，Agent 与工具的通信协议 |
| **A2A** | Agent-to-Agent 协议，Agent 间协作的标准 |
| **Sandbox** | 隔离的执行环境，Agent 用来跑代码/工具 |
| **gVisor** | Google 的用户态内核容器运行时，syscall 拦截 |
| **Kata Containers** | 基于轻量 VM 的容器运行时 |
| **DRA** | Dynamic Resource Allocation，K8s 的细粒度资源申请机制 |
| **InferencePool** | Gateway API Inference Extension 中表示一组推理后端 |
| **Prefix Cache 亲和** | 让相同 prompt 前缀的请求落到同一推理实例，减少重复计算 |
| **Kueue** | K8s 批调度与配额队列 |
| **KEDA** | 事件驱动扩缩容 |
| **Prompt Injection** | 通过输入内容诱导 Agent 执行非预期操作 |

> ⚠️ 复核清单（联网后请确认）：Agent Sandbox 的仓库与 API、Envoy AI Gateway 的成熟度、kagent 是否已进 CNCF Sandbox、Gateway API Inference Extension 的 GA 状态、DRA 的版本状态、A2A 协议归属。
