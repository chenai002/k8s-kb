# AI Agent on Kubernetes 一页纸速览

> 面试前 3 分钟扫一眼。9 个项目全部落在一张分层图里。
> 详细版见 [`ai-agents/README.md`](./ai-agents/README.md)，全景见 [`ai-agent-ecosystem.md`](./ai-agent-ecosystem.md)。

---

## 一、分层图（背这张）

```
┌──────────────────────────────────────────────────────────────────────┐
│ 用户 / 业务系统 / 告警平台                                            │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────────────────┐
│ ① 入口层：AI Gateway                                                  │
│    · 鉴权、token 限额、语义缓存、多模型路由、成本统计                  │
│    · Envoy AI Gateway（代理层） / LiteLLM（应用层）                    │
│    · 文件：ai-agents/ai-gateway.md                                    │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
┌───────────────────────────────▼──────────────────────────────────────┐
│ ② Agent 层：Agent 也是 K8s 资源                                       │
│    · Agent / ModelConfig / ToolServer（CRD）                          │
│    · A2A 协议做 Agent 间协作                                          │
│    · kagent                                                           │
│    · 文件：ai-agents/kagent.md                                        │
└──────┬────────────────────────────────────────────┬──────────────────┘
       │ 调工具                                      │ 调模型
       ▼                                            ▼
┌──────────────────────────┐        ┌───────────────────────────────────┐
│ ③ 工具层：MCP             │        │ ④ 推理路由层                       │
│    · MCP Server for K8s   │        │    · InferencePool + EPP          │
│    · 只读 SA + RBAC       │        │    · 队列长度 / KV Cache / 前缀亲和 │
│    · 文件：mcp-server-k8s │        │    · 文件：inference-extension.md  │
└──────────┬───────────────┘        └───────────────┬───────────────────┘
           │                                        │
           ▼                                        ▼
┌──────────────────────────┐        ┌───────────────────────────────────┐
│ ⑤ 执行层：沙箱             │        │ ⑥ 模型服务层                       │
│    · Agent Sandbox        │        │    · KServe 部署 + vLLM 推理       │
│    · 预热池 / 快照         │        │    · PagedAttention + 连续批处理   │
│    · gVisor / Kata 隔离   │        │    · 文件：kserve-vllm.md          │
│    · 文件：agent-sandbox  │        └───────────────┬───────────────────┘
│             gvisor-kata   │                        │
└──────────────────────────┘                        │
                                                    │
┌───────────────────────────────────────────────────▼──────────────────┐
│ ⑦ 资源与底座                                                          │
│    · GPU Operator（驱动/插件/监控） + DRA（细粒度分配/MIG/拓扑）       │
│    · Kueue（配额排队、借用、防碎片）                                   │
│    · gVisor / Kata（隔离运行时）                                       │
│    · 文件：gpu-operator-dra.md、kueue.md                              │
└──────────────────────────────────────────────────────────────────────┘
```

**一句话串起来**：
> 请求进 AI Gateway → 交给 kagent 上的 Agent → Agent 用 MCP 工具查 K8s、用沙箱跑代码 → 需要模型时走 InferencePool 路由到 vLLM → 底层由 GPU Operator/DRA 供卡、Kueue 管配额。

---

## 二、9 个项目速查卡

| # | 项目 | 一句话 | 解决的核心矛盾 | 关键技术点 |
| --- | --- | --- | --- | --- |
| 1 | **MCP Server for K8s** | 给 AI 一只操作集群的手 | Agent 要操作 K8s，但不能给它 root shell | MCP 协议、JSON Schema、JSON-RPC、ServiceAccount |
| 2 | **kagent** | Agent 变成 K8s 资源 | Agent 配置不可共享/不可治理 | CRD + Controller、Agent Loop、A2A |
| 3 | **gVisor / Kata** | 更强的容器隔离 | Agent 执行不可信代码 | 用户态内核拦截 syscall / 轻量 VM |
| 4 | **Agent Sandbox** | 秒开的隔离执行环境 | Pod 创建太慢、数量压力大 | 预热池、快照恢复、RuntimeClass |
| 5 | **AI Gateway** | Agent 调模型的总入口 | Key 散落、预算失控、无法切模型 | token 限流、语义缓存、SSE 流式、Provider 抽象 |
| 6 | **KServe + vLLM** | 把模型跑成 K8s 服务 | 自建模型吞吐低、难运维 | PagedAttention、连续批处理、CRD 部署、缩到 0 |
| 7 | **Inference Extension** | 请求发给哪个实例最划算 | 轮询导致队头阻塞、缓存不命中 | InferencePool、EPP、ext_proc、前缀感知 |
| 8 | **Kueue** | 任务排队领 GPU | 资源碎片、配额不公平 | suspend 准入、Cohort 借用、ResourceFlavor |
| 9 | **GPU Operator + DRA** | 让 K8s 认识并分细 GPU | 整数分配浪费、拓扑无法表达 | Device Plugin、GFD 标签、ResourceClaim + CEL |

---

## 三、最容易答错的 8 个点

| 常见错误说法 | 正确说法 |
| --- | --- |
| “K8s 官方有 MCP Server” | ❌ 没有。现有实现都是社区/第三方 |
| “用了 Kueue 就不用调度器了” | ❌ Kueue 只管「能不能开始」，kube-scheduler 管「放哪台机器」 |
| “Agent Sandbox 和 gVisor 是竞品” | ❌ 上下游关系：Sandbox 管创建，gVisor 管隔离 |
| “装了 RequestAuthentication 就强制要 token” | ❌（Istio 场景）它只校验，拒绝要靠 AuthorizationPolicy |
| “GPU 可以申请 0.5 张” | ❌ 传统 `nvidia.com/gpu` 只能整数；细粒度用 MIG/DRA |
| “Inference Extension 替代 K8s Service” | ❌ 它是路由层扩展，底层仍依赖服务发现 |
| “vLLM 会自动扩缩容” | ❌ 扩缩是 KServe/HPA/KEDA 的事 |
| “kagent 是 CNCF 官方项目” | ⚠️ 是开源社区项目，官方归属需核实 |

---

## 四、面试自问自答（30 秒版）

**Q：K8s 原生支持 AI Agent 吗？**
不支持。Agent 需要「长会话 + 快速隔离沙箱 + 工具治理」，K8s 的抽象是无状态服务/有状态集。社区在用 Agent Sandbox、RuntimeClass、Gateway API 扩展来补齐。

**Q：为什么 Agent 执行代码要用 gVisor/Kata？**
代码由模型生成，不可信。普通容器共享内核，逃逸即控节点。gVisor 用用户态内核拦截 syscall，Kata 用轻量 VM 给独立内核。代价是启动慢/兼容性/无 GPU 直通。

**Q：推理负载的调度和 Web 有什么不同？**
请求耗时差异大（队头阻塞）、前缀缓存有亲和收益、需要感知队列与 KV Cache。所以有了 InferencePool + EPP，而不是轮询。

**Q：AI Gateway 和普通网关的区别？**
计量单位是 token 不是请求；响应是流式；耗时是分钟级；需要语义缓存与多 provider 降级。

**Q：如何保证 Agent 操作集群安全？**
最小权限 SA（默认只读）、工具粒度授权、写操作人工确认、危险资源拉黑、审计日志、限制返回数据量。

---

## 五、按场景选型（30 秒决策树）

```
你要做什么？
│
├─ 让运维用自然语言查集群
│   └─▶ MCP Server + 只读 SA（最轻）
│
├─ 让 Agent 自动巡检/排障/出报告
│   └─▶ kagent（Agent 即资源，可调度、可治理）
│
├─ Agent 要执行生成的代码
│   └─▶ Agent Sandbox + gVisor（量大） / Kata（要兼容性）
│
├─ 自建模型推理
│   └─▶ GPU Operator + KServe + vLLM（+ DRA 细粒度）
│
├─ 模型有多个实例，想提吞吐
│   └─▶ Inference Extension（InferencePool + EPP）
│
├─ 多个团队抢 GPU
│   └─▶ Kueue（配额 + 借用 + 排队）
│
└─ 多个 Agent 调多家模型，要控成本
    └─▶ AI Gateway（Envoy AI Gateway / LiteLLM）
```

---

## 六、记忆口诀

> **手、人、墙、屋、门、机、导、队、卡**

| 字 | 对应 |
| --- | --- |
| 手 | MCP Server（手：操作集群） |
| 人 | kagent（人：会干活的 Agent） |
| 墙 | gVisor / Kata（墙：隔离） |
| 屋 | Agent Sandbox（屋：秒开的执行环境） |
| 门 | AI Gateway（门：进模型的门） |
| 机 | KServe + vLLM（机：推理的发动机） |
| 导 | Inference Extension（导：把请求导到对的实例） |
| 队 | Kueue（队：排队领卡） |
| 卡 | GPU Operator + DRA（卡：GPU 本身） |

---

> ⚠️ 全部内容基于截止 2026 年中的知识，**当前无法联网核实**。落地前请以官方仓库为准。
