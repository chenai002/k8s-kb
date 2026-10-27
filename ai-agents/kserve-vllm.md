# KServe 与 vLLM（把大模型当成一个 K8s 服务跑起来）

> ⚠️ **待核实**：KServe 的 API 版本（`ServingRuntime` / `LLMInferenceService` 等）演进较快，vLLM 参数名也可能变化，请以官方文档为准。

| 项 | 内容 |
| --- | --- |
| 方向 | A. 在 K8s 上跑 Agent 的「模型后端」 |
| 难度 | ⭐⭐⭐ |
| 是什么 | KServe = 模型服务框架；vLLM = 高吞吐推理引擎 |
| 一句话 | vLLM 是「发动机」，KServe 是「把发动机装进车里并管好它」 |

---

## 一、先看一个场景

Agent 的「大脑」是一个大模型。你有三个选择：

| 方案 | 优点 | 缺点 |
| --- | --- | --- |
| 直接调 OpenAI / 云厂商 API | 零运维 | 贵、数据出境、不可控 |
| 自己用 `python app.py` 起个服务 | 简单 | 没有并发、没有扩缩、没有 GPU 利用率 |
| **KServe + vLLM** | 高性能、可扩缩、可监控 | 需要 K8s 与 GPU 基础 |

大多数「自建模型」的团队最终会走到第三条路。

---

## 二、先搞懂 vLLM

### 2.1 vLLM 解决什么问题

大模型推理有两个瓶颈：

1. **显存不够**：模型参数 + KV Cache（对话历史缓存）占满显存。
2. **吞吐太低**：一个请求算完才处理下一个，GPU 利用率低。

vLLM 的核心创新是 **PagedAttention**。

### 2.2 PagedAttention（重点，面试常问）

**问题**：传统方案给每个请求预留一段连续显存存 KV Cache。但因为「预留长度 = 最大长度」，实际用不到那么多，导致**显存浪费 60–80%**。

**vLLM 的做法**：借鉴操作系统的**虚拟内存分页**：

```
传统：
  [请求A 预留 2048 格，实际用 300]  [请求B 预留 2048 格，实际用 100]
  → 大量浪费

vLLM PagedAttention：
  逻辑块表                物理块（不连续）
  请求A: [块3, 块7, 块9] ──▶ [块3][块7][块9] ...
  请求B: [块4]           ──▶ [块4]
  → 按需分配，几乎无浪费；相同前缀还能共享物理块
```

好处：
- 显存利用率大幅提升 → **能跑更大的 batch** → 吞吐提升（官方数据可提升数倍）。
- **前缀共享**：多个请求有相同系统提示时，物理块可共享（prefix caching）。

### 2.3 连续批处理（Continuous Batching）

传统批处理：等一批请求齐了再一起算，短的等长的。
vLLM：**每一轮迭代都重新组 batch**，新请求可以随时插入，完成请求立即退出。

结果：GPU 几乎不空闲，吞吐高。

### 2.4 vLLM 长什么样

```bash
vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --port 8000 \
  --tensor-parallel-size 1 \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.90
```

启动后暴露 **OpenAI 兼容接口**：

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"meta-llama/Llama-3.1-8B-Instruct","messages":[{"role":"user","content":"hi"}]}'
```

这意味着：**Agent 用 OpenAI SDK 就能调本地模型**，零改造。这也是为什么很多 AI Gateway 要求「统一 OpenAI 格式」。

---

## 三、再搞懂 KServe

### 3.1 它做了什么

如果只有 vLLM，你要自己写 Deployment、Service、HPA、探针、模型下载逻辑…… KServe 帮你做这些：

| 能力 | 说明 |
| --- | --- |
| **CRD 声明式部署** | 写一个 `InferenceService` 就能起模型服务 |
| **多框架支持** | vLLM、Triton、HuggingFace、TorchServe、TFServing 等 |
| **Serverless 缩容到 0** | 没流量时缩到 0，省 GPU（依赖 Knative） |
| **模型存储抽象** | 从 S3 / PVC / OCI 拉模型 |
| **预处理/后处理** | 支持 `transformer` 与 `predictor` 两段式 |
| **Canary 发布** | 新旧模型按比例分流 |
| **标准协议** | V1/V2 推理协议，统一入口 |

### 3.2 核心对象

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: llama-chat
  namespace: ai
spec:
  predictor:
    minReplicas: 1
    maxReplicas: 4
    model:
      modelFormat: { name: huggingface }     # 或 vllm 等
      storageUri: "s3://models/llama-3.1-8b"
      resources:
        limits: { nvidia.com/gpu: "1" }
```

> ⚠️ 面向 LLM 的场景，KServe 也在推进专门的对象（如 `LLMInferenceService`）以更好支持 vLLM 参数与路由，具体以官方为准。

### 3.3 更大的定位：KServe 只是「推理」这一层

```
┌────────────────────────────────────────────────┐
│ 用户 / Agent                                    │
└─────────────────┬──────────────────────────────┘
                  ▼
┌────────────────────────────────────────────────┐
│ AI Gateway（鉴权、限额、缓存） 见 ai-gateway.md │
└─────────────────┬──────────────────────────────┘
                  ▼
┌────────────────────────────────────────────────┐
│ KServe / InferencePool（找哪个实例）             │
└─────────────────┬──────────────────────────────┘
                  ▼
┌────────────────────────────────────────────────┐
│ vLLM Pod（GPU 上真正算的进程）                   │
└────────────────────────────────────────────────┘
                  ▲
                  │
        GPU Operator / DRA（见 gpu-operator-dra.md）
```

---

## 四、实现原理（拆开看）

### 4.1 一个模型服务起不来，通常是这几步出错

KServe 的启动流程：

```
① 你 apply InferenceService
② KServe controller 校验、生成 Knative Service / Deployment
③ Pod 启动：
   ├─ storage-initializer（init 容器）下载模型
   │    s3://... → /mnt/models
   ├─ 主容器加载模型到 GPU
   │    （大模型加载 30s ~ 几分钟）
   └─ 就绪探针通过
④ Service / Gateway 开始路由
```

**常见坑就在这里**：
- 模型下载失败（S3 权限、内网不通）；
- 模型太大超出显存；
- 加载时间超过探针超时 → 反复重启；
- 没有 GPU 节点可用 → Pending。

### 4.2 缩容到 0 与「冷启动」

Serverless 缩到 0 很省 GPU，但**冷启动极慢**（加载 8B 模型可能 1–3 分钟）。所以：

| 策略 | 适合 |
| --- | --- |
| `minReplicas: 0` | 低频、可接受等待 |
| `minReplicas: 1` | 在线服务（推荐） |
| 保留 `minReplicas` + HPA 按 QPS/GPU 利用率扩 | 流量波动大 |

### 4.3 GPU 怎么被分配

K8s 里 GPU 是 **扩展资源**：

```yaml
resources:
  limits:
    nvidia.com/gpu: "1"     # requests 与 limits 必须相等
```

调度器看到这个，就只在有 GPU 的节点上调度。细节见 [gpu-operator-dra.md](./gpu-operator-dra.md)。

### 4.4 大模型需要多卡

一个 70B 模型单卡放不下，vLLM 用 **张量并行（TP）**：

```bash
--tensor-parallel-size 4    # 用 4 张卡切分模型
```

这时一个 Pod 要申请 4 张 GPU。调度器需要能**一次性分配 4 张**（gang 语义），否则可能分配到不在一台机器上的卡。

> 这是 DRA 与 Kueue 要解决的问题之一。

---

## 五、怎么用（最小流程）

### 5.1 前提

```bash
# 1. 有 GPU 节点 + NVIDIA device plugin
kubectl get nodes -l nvidia.com/gpu.present=true
kubectl describe node <gpu-node> | grep nvidia.com/gpu

# 2. 装 KServe（通常用 Helm / kustomize）
#    以官方快速开始为准

# 3. 准备模型（S3 / PVC / 已内置在镜像里）
```

### 5.2 直接跑 vLLM（不装 KServe 也行）

最简单的入门方式：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm
  namespace: ai
spec:
  replicas: 1
  selector:
    matchLabels: { app: vllm }
  template:
    metadata:
      labels: { app: vllm }
    spec:
      containers:
        - name: vllm
          image: vllm/vllm-openai:latest
          args:
            - "--model"
            - "Qwen/Qwen2.5-7B-Instruct"
            - "--max-model-len"
            - "8192"
            - "--gpu-memory-utilization"
            - "0.9"
          ports:
            - containerPort: 8000
          resources:
            limits:
              nvidia.com/gpu: "1"
          # 大模型加载慢，探针要给足时间
          startupProbe:
            httpGet: { path: /health, port: 8000 }
            failureThreshold: 60
            periodSeconds: 10
          readinessProbe:
            httpGet: { path: /health, port: 8000 }
---
apiVersion: v1
kind: Service
metadata:
  name: vllm
  namespace: ai
spec:
  selector: { app: vllm }
  ports: [{ port: 8000, targetPort: 8000 }]
```

### 5.3 用 KServe 的方式

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: qwen-chat
  namespace: ai
spec:
  predictor:
    minReplicas: 1
    maxReplicas: 3
    containers:
      - name: kserve-container
        image: vllm/vllm-openai:latest
        args: ["--model", "Qwen/Qwen2.5-7B-Instruct"]
        resources:
          limits: { nvidia.com/gpu: "1" }
```

**对比**：KServe 版多了自动扩缩、canary、模型存储抽象、统一网关。

---

## 六、有什么坑

### 6.1 显存规划

模型参数量 vs 显存：

| 模型 | FP16 显存（约） | 建议卡 |
| --- | --- | --- |
| 7B | 14 GB + KV Cache | 1 × 24GB |
| 13B | 26 GB | 1 × 40GB 或 TP=2 |
| 70B | 140 GB | TP ≥ 4 × 80GB |

还要留 `gpu-memory-utilization`（默认 0.9）给 KV Cache。

### 6.2 启动慢导致探针杀 Pod

8B 模型加载可能 1 分钟以上。**必须用 `startupProbe`**，否则 liveness 会不断杀掉正在加载的 Pod（表现为无限重启）。

### 6.3 并发与超时

推理是长请求。Service/网关的默认超时（30s/60s）要调大；vLLM 的 `--max-num-seqs` 决定并发批次大小。

### 6.4 节点碎片

一个 8 卡节点跑了 7 个 1 卡 Pod，剩下 1 张卡放不下 4 卡模型 → 资源浪费。
解决：用 Kueue 配额 + gang 调度，或节点池按规格划分。

### 6.5 缩容到 0 的代价

如前所述，冷启动极慢。在线服务不要缩到 0。

### 6.6 模型许可与合规

HuggingFace 上的模型许可证各不相同，商用前要确认。

---

## 七、小白容易混淆的点

| 混淆 | 澄清 |
| --- | --- |
| **KServe 是模型吗？** | 不是，它只是「把模型服务部署起来」的框架 |
| **vLLM 和 Ollama 的区别？** | Ollama 面向本地单机体验；vLLM 面向生产高吞吐、多并发、多卡 |
| **vLLM 和 Triton 的关系？** | 都是推理服务器，Triton 支持多框架（含非 LLM），vLLM 专精 LLM 高吞吐 |
| **KServe 和 Knative 什么关系？** | KServe 早期重度依赖 Knative 做 Serverless；也支持不用 Knative 的模式 |
| **用了 vLLM 就能自动扩缩吗？** | 不能，扩缩是 KServe/HPA/KEDA 的事 |
| **模型放哪里？** | 可以打进镜像（简单但不灵活）、挂 PVC、或用对象存储 + storage-initializer |

---

## 八、面试会怎么问

**Q1：vLLM 为什么快？**
PagedAttention（分页管理 KV Cache，减少显存浪费、支持前缀共享）+ 连续批处理（动态组 batch，GPU 不空转）。

**Q2：KV Cache 是什么？**
推理时缓存已算过的 Key/Value，避免重复计算。对话越长占用越大，是显存瓶颈的主要来源。

**Q3：如何把一个 70B 模型跑起来？**
需要张量并行（TP）跨多卡，Pod 申请多张 GPU；注意调度要保证卡在同一节点、同一 NVLink 域。

**Q4：KServe 的核心价值？**
把模型服务标准化：声明式部署、多框架、缩容到 0、存储抽象、灰度、统一协议。

**Q5：模型加载慢怎么办？**
startupProbe 放宽、模型预拉到节点（或本地 PV）、使用快照、提高 `minReplicas` 避免冷启动。

---

## 九、延伸阅读

- vLLM 官方文档（PagedAttention 论文）
- KServe 官方文档 ⚠️
- 同目录：[inference-extension.md](./inference-extension.md)
- 同目录：[ai-gateway.md](./ai-gateway.md)
- 同目录：[gpu-operator-dra.md](./gpu-operator-dra.md)
