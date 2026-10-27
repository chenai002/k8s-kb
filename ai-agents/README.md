# AI Agent 项目入门（小白版）

> 这里把「Kubernetes 社区里跟 AI Agent 相关的项目」一个一个拆开讲。
> 每篇都按同一个模板写：**它是干什么的 → 为什么需要它 → 它是怎么实现的（原理） → 怎么用 → 有什么坑 → 小白容易混淆的点**。
> 没有 AI 或 K8s 基础也能读，因为每篇都会先把背景讲清楚。

---

## 先看这一篇：两个方向别搞混

在开始之前，先建立一个最基本的认知。社区里说「K8s + AI Agent」，其实在说两件完全不同的事：

### 方向 A：**在** K8s 上运行 Agent

> Agent（比如能自己写代码、调工具的 AI 程序）需要一个地方跑。K8s 是最好的「跑程序的地方」，但 Agent 有些特殊需求，K8s 原生支持得不好，所以要补一些组件。

例子：Agent Sandbox（给它一个安全的执行环境）、KServe（跑模型）、推理网关（统一接模型）。

### 方向 B：**用** Agent 来管理 K8s

> 反过来，让 AI 帮你操作集群，比如「帮我看看哪个 Pod 在报错」。

例子：MCP Server（给 AI 一个操作集群的「手」）、kagent（把 Agent 变成 K8s 里的一个资源）。

### 方向 C：底座能力

> 不管哪个方向，都需要 GPU、调度、安全隔离、监控这些基础设施。

例子：GPU Operator、Kueue、gVisor、DRA。

**记住这张表，后面每篇文章都会告诉你它属于哪个方向。**

---

## 文档索引

### 方向 A：在 K8s 上跑 Agent

| # | 文档 | 一句话 | 难度 |
| --- | --- | --- | --- |
| 1 | [agent-sandbox.md](./agent-sandbox.md) | 给 Agent 一个「秒开、用完就扔、互相隔离」的代码执行环境 | ⭐⭐⭐ |
| 2 | [gvisor-kata.md](./gvisor-kata.md) | Agent 要执行 AI 生成的代码，怎么防止它把宿主机搞坏 | ⭐⭐ |
| 3 | [kserve-vllm.md](./kserve-vllm.md) | 把大模型当成一个 K8s 服务跑起来，并支持自动扩缩 | ⭐⭐⭐ |
| 4 | [inference-extension.md](./inference-extension.md) | 让网关「聪明地」把请求分给最合适的模型实例 | ⭐⭐⭐⭐ |
| 5 | [ai-gateway.md](./ai-gateway.md) | Agent 调模型的总入口：鉴权、限额、缓存、多模型切换 | ⭐⭐ |
| 6 | [mcp-server-k8s.md](./mcp-server-k8s.md) | 给 AI 装一只「能操作 K8s 的手」，但要装锁 | ⭐⭐ |

### 方向 B：用 Agent 运维 K8s

| # | 文档 | 一句话 | 难度 |
| --- | --- | --- | --- |
| 7 | [kagent.md](./kagent.md) | 把 Agent 定义成 K8s 资源（YAML），让它住在集群里干活 | ⭐⭐⭐ |
| 8 | [kueue.md](./kueue.md) | 大量 AI 任务排队领 GPU，防止互相抢 | ⭐⭐ |

### 方向 C：底座能力

| # | 文档 | 一句话 | 难度 |
| --- | --- | --- | --- |
| 9 | [gpu-operator-dra.md](./gpu-operator-dra.md) | 让 K8s 认识 GPU，并能切得很细地分配 | ⭐⭐⭐ |

### 其他

| # | 文档 | 说明 |
| --- | --- | --- |
| — | [../ai-agent-ecosystem.md](../ai-agent-ecosystem.md) | 全景概览与选型建议 |
| — | [../ai-agents-cheatsheet.md](../ai-agents-cheatsheet.md) | **一页纸速览**（分层图 + 速查卡 + 易错点 + 口诀） |

---

## 建议阅读顺序（给完全新手）

```
1. mcp-server-k8s.md    ← 最容易理解，先建立「AI 怎么操作工具」的概念
2. kagent.md            ← 看「Agent 变成 K8s 资源」是什么样
3. gvisor-kata.md       ← 理解「隔离」这件事
4. agent-sandbox.md     ← 把上面两个概念合起来
5. ai-gateway.md        ← 理解 Agent 怎么调模型
6. kserve-vllm.md       ← 模型怎么跑起来
7. inference-extension.md ← 模型请求怎么被智能调度
8. kueue.md             ← 资源怎么排队
9. gpu-operator-dra.md  ← 最底层的资源管理
```

---

## 重要说明

> **本系列文档基于我掌握的知识编写（截止 2026 年中的公开信息），当前环境无法联网核实。**
> AI 领域的项目迭代非常快，版本号、是否 GA、API 名称都可能已经变化。
> 每篇文档顶部都有「⚠️ 待核实」提示，**落地前请务必以官方仓库/文档为准**。

## 通用术语速查

| 术语 | 一句话解释 |
| --- | --- |
| **Agent** | 一个会用大模型思考、会调用工具、能自己一步步完成任务的程序 |
| **LLM** | 大语言模型，Agent 的「大脑」 |
| **Tool** | Agent 能调用的工具，比如「查询 Pod 列表」这个函数 |
| **MCP** | Model Context Protocol，Agent 和工具之间说话的「通用语」 |
| **沙箱（Sandbox）** | 一个隔离的执行环境，里面搞坏了也影响不到外面 |
| **CRD** | Custom Resource Definition，K8s 的「自定义资源类型」，让你能 `kubectl get 自己的类型` |
| **Operator** | 一个「会自己干活的控制器」，盯着 CRD 的状态并把它变成现实 |
| **Pod** | K8s 里最小的运行单位，一个或多个容器的组合 |
| **RuntimeClass** | 告诉 K8s「这个 Pod 用什么容器运行时」，比如 gVisor |
| **gVisor / Kata** | 两种「比普通容器更隔离」的运行时 |
| **推理（Inference）** | 用训练好的模型回答问题的过程 |
| **GPU** | 显卡，跑大模型推理需要它 |
| **token** | 模型处理文本的基本单位，也是计费单位 |
