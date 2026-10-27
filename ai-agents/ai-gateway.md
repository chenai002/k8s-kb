# AI Gateway（Agent 调模型的总入口）

> ⚠️ **待核实**：本文基于截止 2026 年中的公开知识整理，**无法联网核对**。Envoy AI Gateway、LiteLLM 等项目迭代很快，功能与 API 请以官方文档为准。

| 项 | 内容 |
| --- | --- |
| 方向 | A. 在 K8s 上跑 Agent |
| 难度 | ⭐⭐ |
| 是什么 | 专门为「调用大模型」设计的网关层 |
| 一句话 | 所有 Agent 调模型都走它，它负责鉴权、限额、缓存、选路 |

---

## 一、先看一个场景

你的团队里有 20 个 Agent：客服、代码审查、日志分析、周报生成……
它们各自都在调模型（OpenAI / 自建 vLLM / 国产模型）。

于是出现这些事：

- 每个人的 API Key 散落在各个 Pod 的环境变量里，**泄露一个就全暴露**。
- 某个 Agent 写了个死循环，一夜之间刷了 **3000 美元**。
- 想做模型对比：「A 模型便宜、B 模型准」，但没有统一的地方切。
- 出问题查不到：**哪个 Agent、用了多少 token、延迟多少**。
- 相同的问题被问了 1 万次，每次都重新算，**白烧钱**。

**AI Gateway 就是解决这些问题的。** 它位于 Agent 和模型之间，像公司统一的「报销与采购中心」。

---

## 二、它具体做了什么

### 2.1 核心能力清单

| 能力 | 解决什么 | 一句话原理 |
| --- | --- | --- |
| **统一接入 / Provider 抽象** | 各模型 API 长得不一样 | 都转成 OpenAI 兼容格式 |
| **密钥托管** | Key 到处散落 | Agent 只拿网关的 key |
| **限流与配额** | 某 Agent 刷爆预算 | 按 token 数或请求数限流 |
| **路由/权重分流** | 新旧模型灰度、降级 | 按 header/权重选后端 |
| **语义缓存** | 重复问题重复算 | 用向量相似度命中缓存 |
| **故障转移** | 某个 provider 挂了 | 自动重试到备用 provider |
| **可观测** | 花在哪、多慢、错多少 | 记录 token/延迟/错误码 |
| **请求/响应改写** | 注入系统提示、脱敏 | 过滤器改写 body |

### 2.2 和普通 API 网关的区别

| | 普通网关（如 Nginx Ingress） | AI Gateway |
| --- | --- | --- |
| 流量单位 | 请求数 / QPS | **token 数**（输入+输出） |
| 计费 | 按请求 | **按 token，且输入输出价格不同** |
| 缓冲 | 一般 | 流式响应（SSE）要**逐块转发** |
| 超时 | 秒级 | **可能几十秒到几分钟**（长生成） |
| 缓存 | URL + header | **语义相似度** |
| 后端协议 | HTTP | OpenAI Chat Completions、Embeddings 等 |

**最关键的一条**：LLM 的限流不能只按 QPS。一个请求可能 10 个 token，也可能 10 万个。所以必须 **按 token 限流与计量**。

---

## 三、实现原理（拆开看）

### 3.1 位置

```
┌─────────┐   ┌─────────┐   ┌─────────┐
│ Agent A │   │ Agent B │   │ Agent C │
└────┬────┘   └────┬────┘   └────┬────┘
     └─────────────┼─────────────┘
                   ▼
        ┌─────────────────────┐
        │    AI Gateway       │   ← 本篇
        │ 鉴权 / 限额 / 缓存   │
        │ 路由 / 观测 / 改写   │
        └──────┬──────┬───────┘
               ▼      ▼
        ┌──────────┐ ┌──────────────────┐
        │ OpenAI   │ │ 自建 vLLM / 其他  │
        └──────────┘ └──────────────────┘
```

### 3.2 一次请求的处理链

```
① 认证
   · 检查 Agent 的 API Key（网关自己的 key，不是上游 key）
   · 识别出「这是哪个 Agent / 哪个团队」

② 配额检查
   · 查该团队本月已用 token
   · 超了就返回 429

③ 缓存查找
   · 计算 prompt 的 embedding
   · 在向量库中找相似度 > 阈值的缓存结果
   · 命中 → 直接返回（省钱省时间）

④ 路由决策
   · 按模型名 / header / 权重映射到具体 provider
   · 选择后端实例（可能是自建 vLLM 集群的某个实例）

⑤ 请求改写
   · 注入系统提示、加 trace header、脱敏

⑥ 转发（流式）
   · 用 SSE 流式转发响应，边收边发

⑦ 计量与记录
   · 统计输入/输出 token（从响应的 usage 字段或自己算）
   · 记 Prometheus 指标、访问日志

⑧ 返回
```

### 3.3 关键机制讲解

#### ① Provider 抽象

各家 API 的字段名、错误码、流式格式都不同。网关做**协议转换**：

```
Agent 发的是 OpenAI 格式
  │
  ├─▶ OpenAI：直接转发
  ├─▶ vLLM：直接转发（vLLM 兼容 OpenAI 格式）
  └─▶ 其他厂商：网关做字段映射
```

这就是为什么很多网关会要求「统一用 OpenAI 兼容接口」。

#### ② Token 计量

token 怎么来的？
- **响应里带**：大多数 API 返回 `usage.prompt_tokens` / `completion_tokens`。
- **自己算**：用 tokenizer（如 tiktoken）本地计算。
- **流式场景**：必须在最后一块里读取 usage，或自己累加。

**难点**：如果客户端中途断开，上游可能还在跑，token 已经消耗了。所以计量要**在网关侧闭环**，不能依赖客户端。

#### ③ 语义缓存

普通缓存是键值完全匹配；语义缓存是「意思相近就复用」。

```
"如何重启 Pod？"        ─┐
"怎么重启一个 Pod"      ─┼──▶ embedding 相似度 0.95 → 命中同一条缓存
"Pod 重启的方法是什么"  ─┘
```

风险：**相似但实际不同的问题会答错**（比如「重启 Prod 的 Pod」和「重启 Dev 的 Pod」）。所以通常：
- 只在低风险场景开启；
- 阈值设高（0.95+）；
- 缓存键里带上租户/模型/温度等参数。

#### ④ 流式转发（SSE）

LLM 是逐字生成的，网关必须支持 **Server-Sent Events 流式转发**，否则用户要等几十秒才看到第一个字。

技术点：
- 不能用传统的「缓冲完整响应再返回」；
- 要处理上游断开、客户端断开；
- 中途断开的 token 仍要计量。

---

## 四、对比：两类实现方案

### 4.1 Envoy AI Gateway（基于 Envoy + Gateway API）

**思路**：在通用代理 Envoy 之上，增加「LLM 感知」的扩展点。

```
Gateway API（HTTPRoute）
       │
       ▼
Envoy（数据面，见 ../envoy.md）
  └─ AI 相关的 filter / ext_proc
       · 解析请求体，提取 model / prompt
       · 调用外部限流服务做 token 配额
       · 语义缓存集成
       ▼
上游：OpenAI / vLLM / 其他
```

**优点**：
- 复用 Envoy 的成熟能力（TLS、可观测、LB）。
- 与 K8s Gateway API 一致，符合平台化趋势。
- 性能好。

**缺点**：
- 需要在代理层解析 JSON body，复杂度高。
- 生态与文档还在成长。

### 4.2 LiteLLM Proxy（应用层网关）

**思路**：直接用 Python 写一个「LLM 代理服务」，以 Deployment 部署。

```
Agent → LiteLLM Pod（FastAPI）
          · 统一 OpenAI 兼容接口
          · 多 provider 适配
          · 虚拟 key、预算、限流
          · 成本统计
        → 上游 provider
```

**优点**：
- 上手极快，配置 YAML 即可。
- provider 覆盖广。
- 自带成本统计与虚拟 key。

**缺点**：
- Python 应用，性能与并发不如代理层。
- 单点需自己做高可用。

**选型建议**：小团队起步用 LiteLLM；大规模、已在用 Gateway API 的平台用 Envoy AI Gateway。

---

## 五、怎么用（示例）

### 5.1 LiteLLM 风格配置

```yaml
# config.yaml（示意）
model_list:
  - model_name: fast
    litellm_params:
      model: openai/gpt-4.1-mini
      api_key: os.environ/OPENAI_API_KEY
  - model_name: local-llama
    litellm_params:
      model: hosted_vllm/meta-llama/Llama-3.1-8B
      api_base: http://vllm.ai.svc:8000/v1

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY

router_settings:
  routing_strategy: usage-based-routing
  fallbacks: [{"fast": ["local-llama"]}]
```

```yaml
# K8s 部署（示意）
apiVersion: apps/v1
kind: Deployment
metadata: { name: litellm, namespace: ai }
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: litellm
          image: ghcr.io/berriai/litellm:main-latest
          args: ["--config", "/config/config.yaml", "--port", "4000"]
          ports: [{ containerPort: 4000 }]
```

### 5.2 Gateway API 风格（概念）

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: llm-route, namespace: ai }
spec:
  parentRefs: [{ name: ai-gateway }]
  rules:
    - matches:
        - path: { type: PathPrefix, value: /v1/chat }
      backendRefs:
        - name: vllm-pool
          port: 8000
```

### 5.3 Agent 侧

```python
from openai import OpenAI
client = OpenAI(
    base_url="http://ai-gateway.ai.svc/v1",   # 指向网关
    api_key=os.environ["GATEWAY_KEY"],        # 网关发的 key，不是上游 key
)
```

**关键**：Agent 永远不知道上游是什么模型、key 是什么。

---

## 六、有什么坑

### 6.1 token 计量不准

- 流式响应中断 → usage 丢失。
- 不同模型 tokenizer 不同 → 自己算会偏差。
- 工具调用（function calling）的 token 也要算。

**建议**：以 provider 返回的 usage 为准，自己算作为兜底。

### 6.2 超时设置

LLM 生成可能几十秒。网关默认超时（如 60s）会切断长回答。
**要按路由单独设置**，并且区分「首字延迟」与「总时长」。

### 6.3 限流维度

只按 QPS 限流没用。要支持：
- 输入 token / 输出 token 分别计；
- 按团队/租户/虚拟 key；
- 按模型（贵的模型额度小）。

### 6.4 缓存污染

把带用户隐私的 prompt 缓存起来，可能被其他用户命中。
**必须按租户隔离缓存**。

### 6.5 语义缓存的正确性

相似度阈值太低会答错。建议只对**FAQ 类、低风险**场景开启。

### 6.6 密钥管理

网关自己要存上游 key，这成了新的敏感点。用 External Secrets / Vault，不要写进 Git。

---

## 七、小白容易混淆的点

| 混淆 | 澄清 |
| --- | --- |
| **AI Gateway 是模型吗？** | 不是，它不推理，只转发和管理 |
| **它和 Ingress 的关系？** | 它是 L7 网关的一种，专门为 LLM 优化；可以实现为 Gateway API 的扩展，也可以是独立服务 |
| **它和模型服务平台（KServe）的关系？** | KServe 负责「把模型跑起来」，网关负责「把请求送到正确的模型」。通常网关在前，KServe 在后 |
| **它和 MCP 的关系？** | 完全不同：MCP 是 Agent↔工具，AI Gateway 是 Agent↔模型 |
| **为什么不能直接用 OpenAI SDK？** | 能用，但没有统一鉴权、限额、缓存、多模型、成本统计 |
| **它和 API 管理平台（如 Kong）什么关系？** | Kong 等也在加 LLM 能力；本质是同一层的竞争 |

---

## 八、面试会怎么问

**Q1：为什么需要 AI 专用网关？**
LLM 流量的计量单位是 token、响应是流式、耗时是分钟级、成本敏感、需要语义缓存——普通网关的设计假设都不适用。

**Q2：token 限流怎么实现？**
在网关侧解析请求（估算输入 token）与响应（读取 usage），累加到配额计数器；用滑动窗口或令牌桶；对超限返回 429。

**Q3：语义缓存的原理与风险？**
把 prompt 转向量，做相似度检索；风险是相似但语义不同导致错答，需高阈值 + 租户隔离 + 仅低风险场景。

**Q4：如何处理流式响应？**
SSE 边收边转发，不能缓冲整体；要处理中途断开与 usage 统计。

**Q5：网关挂了怎么办？**
多副本 + 健康检查 + 故障转移到直连（降级模式），或者多活网关。

---

## 九、延伸阅读

- Envoy AI Gateway 官方文档 ⚠️
- LiteLLM 官方文档 ⚠️
- 同目录：[kserve-vllm.md](./kserve-vllm.md)（模型怎么跑起来）
- 同目录：[inference-extension.md](./inference-extension.md)（请求怎么选实例）
- [../envoy.md](../envoy.md)（Envoy 基础）
