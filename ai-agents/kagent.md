# kagent（把 AI Agent 变成 Kubernetes 里的一种资源）

> ⚠️ **待核实**：本文基于截止 2026 年中的公开知识整理，当前无法联网核对。kagent 是一个开源项目（由 SoloSolio 发起），是否能列入 CNCF Sandbox、具体 CRD 字段名请以官方仓库为准。

| 项 | 内容 |
| --- | --- |
| 方向 | B. 用 Agent 运维 K8s |
| 难度 | ⭐⭐⭐ |
| 是什么 | 一个用 CRD 定义和运行 AI Agent 的 K8s 平台 |
| 一句话 | 让你像写 Deployment 一样写 Agent 的 YAML |

---

## 一、先看一个场景

上一篇文章里，我们给 AI 装了一只「操作 K8s 的手」（MCP Server）。但你很快会发现新问题：

- 每个运维同事都要在自己电脑上装 MCP Server、配 kubeconfig —— **没法共享**。
- 不同人用的提示词不一样，排查质量参差不齐 —— **没有标准**。
- Agent 能做什么、不能做什么，没有统一管理 —— **没有治理**。
- 想让 Agent 每天早上自动巡检并出报告 —— **没有调度**。

**kagent 的思路：把 Agent 本身变成 K8s 里的一种资源。**

就像你写 `Deployment` 来描述「要跑 3 个 nginx」，你写 `Agent` 来描述「要跑一个会查 K8s 的助手」。

---

## 二、它到底做了什么

kagent 提供三件核心东西：

1. **一套 CRD**：`Agent`、`ModelConfig`、`Tool` / `ToolServer`（具体名字以官方为准）。
2. **一个控制器 + Runtime**：把 CRD 变成真正跑起来的 Agent 进程（Pod）。
3. **一套内置工具**：查 K8s、查 Prometheus、查日志、查 Helm 等，开箱即用。

另外它还支持 **A2A（Agent-to-Agent）协议**，让多个 Agent 互相调用（比如「诊断 Agent」调用「修复 Agent」）。

---

## 三、核心概念（重点）

### 3.1 Agent

一个 `Agent` 资源 = 「一个人格 + 一套工具 + 一个模型」。

```yaml
apiVersion: kagent.dev/v1alpha2        # ⚠️ 版式以官方为准
kind: Agent
metadata:
  name: k8s-troubleshooter
  namespace: kagent
spec:
  description: "Kubernetes 排障助手"
  systemMessage: |
    你是一个资深 SRE。用户描述故障时，先用工具收集事实，
    再给出结论。不要臆测，不要执行删除操作。
  modelConfig: default-model          # 引用 ModelConfig
  tools:
    - type: McpServer                 # 引用一个工具服务器
      mcpServer:
        name: k8s-tools
        kind: ToolServer
        apiGroup: kagent.dev
        toolNames: ["pods_list", "pods_log", "events_list"]
  deployment:
    replicas: 1
    resources: { requests: { cpu: 100m, memory: 256Mi } }
```

**拆开看**：
- `systemMessage`：给模型的「角色设定」，相当于 system prompt。
- `modelConfig`：用哪个模型、在哪儿（OpenAI / Azure / Ollama / vLLM）。
- `tools`：能用哪些工具，**这里的粒度可以精确到单个工具名**（这是治理的关键）。

### 3.2 ModelConfig

把「模型接入信息」单独抽出来，避免每个 Agent 重复写。

```yaml
apiVersion: kagent.dev/v1alpha2
kind: ModelConfig
metadata:
  name: default-model
  namespace: kagent
spec:
  provider: OpenAI            # 或 Anthropic / AzureOpenAI / Ollama / 自建 vLLM
  model: gpt-4.1-mini
  apiKeySecret:
    name: openai-key
    key: api-key
  # baseURL: https://my-gateway/v1   # 走自建网关时配置
```

**为什么这样设计**：换模型只改一个对象，所有 Agent 跟着变；密钥只存一处。

### 3.3 Tool / ToolServer

工具来源有两种：

1. **内置（built-in）**：kagent 自带一组 K8s 相关工具。
2. **MCP Server**：接外部 MCP Server（这正好和上一篇的 MCP Server 衔接）。

```
Agent ──引用──▶ ToolServer（MCP）──▶ MCP Server Pod ──▶ K8s API
```

### 3.4 Agent 之间的关系（A2A）

```
用户 ──▶ 协调 Agent（planner）
              │  A2A 调用
       ┌──────┼──────┐
       ▼      ▼      ▼
   诊断Agent  查询Agent  修复Agent
```

每个 Agent 有自己的「名片」（`/.well-known/agent.json` 之类的 Agent Card），说明自己能干什么、怎么调用。这就是 A2A 协议的核心。

---

## 四、实现原理（拆开看）

### 4.1 整体架构

```
┌───────────────────────────────────────────────────────────────┐
│ kagent-controller（Deployment，K8s 控制器）                    │
│  · watch Agent / ModelConfig / ToolServer                     │
│  · 为每个 Agent 创建 Deployment + Service + ConfigMap          │
│  · 管理 A2A 注册与发现                                          │
└──────────────────────┬────────────────────────────────────────┘
                       │ 创建/更新
                       ▼
┌───────────────────────────────────────────────────────────────┐
│ Agent Pod（每个 Agent 一个）                                    │
│  · kagent runtime（Python/Go 写的 Agent 执行器）                │
│  · 内含：LLM 调用循环、工具调用、会话管理、A2A 服务端           │
└──────┬───────────────────────────┬────────────────────────────┘
       │ 调模型                     │ 调工具（MCP / HTTP）
       ▼                            ▼
   ModelConfig 指向的端点        ToolServer Pod → K8s API
```

### 4.2 从 YAML 到跑起来：发生了什么

```
① 你 kubectl apply -f agent.yaml
② kagent-controller 收到 Agent 的创建事件
③ 控制器渲染出：
     - Deployment（1 个副本，镜像是 kagent 的 agent runtime）
     - Service（让 A2A / UI 能访问）
     - ConfigMap（把 systemMessage、工具列表等打包进去）
④ Agent Pod 启动
⑤ runtime 读取配置：
     - 连 MCP Server 拉取工具清单（tools/list）
     - 连模型端点
     - 暴露 Agent Card 与执行接口
⑥ 用户通过 UI 或 API 发消息 → runtime 进入 Agent Loop
```

**关键洞察**：kagent 没有发明新的运行机制，它只是**把「配置一个 Agent」这件事翻译成 K8s 原生对象**。这也意味着：Agent 的扩缩容、滚动更新、资源限制、网络策略，全部复用 K8s 的能力。

### 4.3 Agent Loop（真正的核心）

无论哪个 Agent 框架，本质都是一个循环：

```
用户消息
  │
  ▼
┌─────────────────────────────┐
│ 1. 把上下文 + 工具清单发给 LLM │
└──────────────┬──────────────┘
               ▼
      ┌────────────────┐
      │ LLM 输出是什么？ │
      └────┬───────┬───┘
     工具调用│       │最终回答
           ▼       ▼
   ┌──────────────┐  ┌──────────┐
   │ 执行工具      │  │ 返回用户  │
   │ 结果塞回上下文 │  └──────────┘
   └──────┬───────┘
          │
          └──▶ 回到第 1 步（最多 N 轮）
```

**小白理解**：LLM 一次只能「说一句话」，但它可以说「我要调用 pods_list 这个工具」。runtime 执行完把结果再喂给它，它就能基于结果继续推理。如此反复，直到它说「我知道了，原因是……」。

### 4.4 会话（Session）存哪

Agent 需要记住上下文。kagent 的做法通常有：
- **内存**：Pod 重启就丢，适合无状态问答。
- **持久化（ConfigMap/PVC/外部 DB）**：适合长期会话。

⚠️ 具体实现需查官方文档。

---

## 五、怎么用（最小流程）

```bash
# 1. 安装 kagent（通常用 Helm）
helm repo add kagent https://...        # 以官方为准
helm install kagent kagent/kagent -n kagent --create-namespace

# 2. 配置模型密钥
kubectl create secret generic openai-key -n kagent \
  --from-literal=api-key="$OPENAI_API_KEY"

# 3. 创建 ModelConfig + Agent
kubectl apply -f agent.yaml

# 4. 看生成的资源
kubectl get agents -n kagent
kubectl get deploy,svc,pod -n kagent

# 5. 打开 UI（kagent 带 Web UI）
kubectl port-forward -n kagent svc/kagent-ui 8080:80
```

**观察点**：`kubectl get agents` 能看到你定义的 Agent —— 这就是「Agent 即资源」的直观体现。

---

## 六、有什么坑

### 6.1 权限与爆炸半径

Agent 用哪个 ServiceAccount？如果它用的 SA 是 `cluster-admin`，那么「让 AI 帮你排障」就等于「给 AI 一个 root shell」。

**正确做法**：
- Agent 的 Pod 用专用 SA，只读权限；
- 需要写操作时，拆出一个「修复 Agent」，单独授权 + 人工确认；
- 用 NetworkPolicy 限制它只能访问需要的东西。

### 6.2 成本失控

Agent 的每一轮循环都会调用模型。一个「帮忙看下集群」的请求，可能触发 10 次工具调用 = 10 次模型调用。**必须有 token 预算与轮数上限。**

### 6.3 不确定性与幻觉

Agent 可能声称「我发现是内存不足」，但其实没查过。防御：
- system prompt 里强制「先调用工具，再下结论」；
- 要求回答中引用工具返回的原始数据；
- 用固定的排查剧本（runbook）约束它。

### 6.4 与 CRD 生态的耦合

kagent 的 CRD 是它自己的，**不是标准**。如果你已经在用别的 Agent 平台，迁移成本不低。这也是社区还在讨论「Agent CRD 应该长什么样」的原因。

### 6.5 多租户

多个团队共用一个 kagent 集群时，Agent 之间的工具、模型配额、命名空间访问要隔离。

---

## 七、小白容易混淆的点

| 混淆 | 澄清 |
| --- | --- |
| **kagent 是 K8s 官方项目吗？** | 不是。它是开源社区项目，不是 kubernetes-sigs 下的 |
| **kagent 和 LangChain / LangGraph 什么关系？** | 层次不同。LangChain 是「写 Agent 的代码库」，kagent 是「在 K8s 里运行和管理 Agent 的平台」。kagent 内部可能用到类似的 Agent 框架 |
| **kagent 和 MCP Server 什么关系？** | MCP Server 提供工具，kagent 提供一个「用工具的 Agent」。kagent 通过引用 ToolServer 来接入 MCP |
| **A2A 和 MCP 有什么区别？** | MCP 是「Agent ↔ 工具」，A2A 是「Agent ↔ Agent」。一个是调用工具，一个是找人帮忙 |
| **Agent 是一个 Pod 吗？** | 在 kagent 里，通常一个 Agent 对应一个 Deployment（可多副本），副本之间需要处理会话一致性 |
| **它能替代运维吗？** | 现阶段更像「助手」：收集信息、给出建议、执行低风险动作。关键决策仍需人 |

---

## 八、面试会怎么问

**Q1：kagent 解决了什么问题？**
把 Agent 的定义、运行、工具、模型、权限都变成 K8s 原生对象，实现标准化与可治理；相比「每人本地跑一个脚本」，它能共享、可审计、可调度。

**Q2：它的实现原理？**
控制器 watch CRD → 渲染 Deployment/Service/ConfigMap → Agent Runtime 启动后拉取工具清单与模型配置 → 进入「LLM + 工具」循环。

**Q3：怎么控制 Agent 的权限？**
Agent 的 Pod 使用最小权限 SA；工具粒度授权（只暴露部分 toolNames）；写操作人工确认；NetworkPolicy 限制出口。

**Q4：A2A 协议的作用？**
让 Agent 之间标准化地互相调用与发现能力，适合把复杂任务拆成多个专职 Agent。

**Q5：和直接用 MCP Server 相比？**
MCP Server 只是工具层；kagent 还解决运行环境、多 Agent 协作、模型管理、UI、调度。复杂度也更高。

---

## 九、延伸阅读

- kagent 官方仓库与文档 ⚠️
- 同目录：[mcp-server-k8s.md](./mcp-server-k8s.md)（工具从哪来）
- 同目录：[agent-sandbox.md](./agent-sandbox.md)（Agent 执行代码的隔离）
- A2A 协议规范 ⚠️
