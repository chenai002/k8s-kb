# Knative 实战案例集

> 用 7 个具体案例把 Knative 的能力一个个落地。
> 每个案例包含：**业务需求 → 为什么适合 Knative → 完整 YAML → 怎么验证 → 注意事项**。
> 前置阅读：[`knative.md`](./knative.md)（概念与实现原理）

---

## 目录

| # | 案例 | 用到的能力 |
| --- | --- | --- |
| 1 | 图片缩略图服务（缩到 0） | Serving、KPA、min-scale |
| 2 | 新版算法灰度发布 | Revision、Route 权重 |
| 3 | 内部测试专用版本（按 Header 固定入口） | Route tag |
| 4 | 夜间批处理 Job（事件触发） | Eventing、Job Sink |
| 5 | 定时爬虫（Cron + 缩到 0） | PingSource |
| 6 | 上传即处理（Kafka 事件驱动） | KafkaSource、Broker/Trigger |
| 7 | 多租户 SaaS 的函数托管 | Serving 多命名空间 + 配额 |

---

## 案例 1：图片缩略图服务（把 0 副本跑到真正可用）

### 1.1 业务需求

电商平台，用户上传商品图后需要生成三种尺寸缩略图：

- 白天高峰期 QPS 约 50；
- 深夜几乎为 0；
- 单张处理耗时约 200ms；
- 预算敏感，希望空闲时不占资源。

### 1.2 为什么适合 Knative

- 流量有明显的「有/无」两态 → 天然适合缩到 0；
- 处理是短请求（200ms）→ 冷启动可以接受（一次冷启动 3s，但深夜本来也没人）；
- 不需要复杂状态 → 可以随便扩缩。

> 对比：如果这是「核心下单接口」，P99 要求 100ms，那就**不该**缩到 0，应该 `min-scale: 2`。

### 1.3 应用代码（示意）

```python
# app.py —— 一个极简的缩略图服务
from flask import Flask, request, jsonify
from PIL import Image
import io, os

app = Flask(__name__)
PORT = int(os.environ.get("PORT", 8080))

@app.get("/healthz")
def health():
    return "ok"

@app.post("/thumbnail")
def thumbnail():
    file = request.files["image"]
    size = tuple(map(int, request.args.get("size", "200x200").split("x")))
    img = Image.open(io.BytesIO(file.read()))
    img.thumbnail(size)
    buf = io.BytesIO()
    img.save(buf, format="JPEG", quality=85)
    return app.response_class(buf.getvalue(), mimetype="image/jpeg")

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=PORT)
```

**Knative 的要求**：容器必须监听 `PORT` 环境变量指定的端口（默认 8080），并且实现一个健康检查端点。

### 1.4 部署 YAML

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
        autoscaling.knative.dev/min-scale: "0"     # 空闲缩到 0
        autoscaling.knative.dev/max-scale: "30"    # 上限
        autoscaling.knative.dev/target: "10"       # 每副本 10 并发
        autoscaling.knative.dev/scale-down-delay: "2m"  # 空闲 2 分钟后才缩
    spec:
      # 可选：限制单副本资源，避免大图打爆
      containerConcurrency: 10                     # 硬并发上限
      timeoutSeconds: 30                           # 单请求超时
      containers:
        - name: thumbnail
          image: registry.example.com/thumbnail:1.2.0
          ports:
            - containerPort: 8080
          resources:
            requests: { cpu: 100m, memory: 256Mi }
            limits:   { cpu: 1,    memory: 512Mi }
          readinessProbe:
            httpGet: { path: /healthz, port: 8080 }
            periodSeconds: 2
          env:
            - name: MAX_UPLOAD_MB
              value: "20"
```

### 1.5 验证

```bash
# 1. 部署后拿到 URL
kubectl get ksvc thumbnail -n media
# NAME        URL                                           LATESTCREATED   LATESTREADY   READY
# thumbnail   http://thumbnail.media.example.com             thumbnail-00001 thumbnail-00001 True

# 2. 确认当前 0 副本
kubectl get pods -n media
# （无 Pod）

# 3. 发一个请求，观察冷启动
time curl -s -o /dev/null -w "%{http_code}\n" \
  -F "image=@test.jpg" "http://thumbnail.media.example.com/thumbnail?size=200x200"

# 4. 立刻查看 Pod 是否起来
kubectl get pods -n media -w

# 5. 看缩容
sleep 180
kubectl get pods -n media          # 又变回 0 个
```

### 1.6 观察点与调优

| 观察 | 说明 |
| --- | --- |
| 第一次请求耗时 2–5s | 冷启动，含调度 + 拉镜像 + 启动 |
| 后续请求 ~200ms | 正常处理时延 |
| 一分钟后 Pod 变成 `queue-proxy` 的 `Terminating` | 缩容中 |
| `kubectl get revision` | 能看到 `thumbnail-00001` |

**调优**：
- 把 `target` 调小（如 5）→ 更早扩容，更少的排队；
- `scale-down-delay: 5m` → 深夜也有零星请求时，避免频繁冷启动；
- 镜像用小基础镜像（alpine/distroless），减少拉取时间。

---

## 案例 2：新版算法灰度发布（Revision + Route 权重）

### 2.1 业务需求

推荐服务要上线「新版排序算法」。要求：

- 先 5% 流量试；
- 观察错误率与点击率；
- 出问题 1 分钟内回滚；
- 全程不重建旧版本。

### 2.2 为什么适合 Knative

Knative 的 **Revision 不可变**，每次发布就是一个新 Revision。**Route 可以在多个 Revision 之间按权重切流**，回滚就是把权重改回 100/0——**不需要重新部署**。

### 2.3 初始版本（v1，全量）

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: recommender
  namespace: shop
spec:
  template:
    metadata:
      name: recommender-v1
      annotations:
        autoscaling.knative.dev/min-scale: "1"     # 核心服务，不缩到 0
        autoscaling.knative.dev/max-scale: "20"
    spec:
      containers:
        - image: registry.example.com/recommender:1.0.0
```

部署后产生 `recommender-00001`，Route 100% 指向它。

### 2.4 发布 v2（先不接流量）

```bash
kubectl patch ksvc recommender -n shop --type=merge -p '
spec:
  template:
    metadata:
      name: recommender-v2
    spec:
      containers:
        - image: registry.example.com/recommender:2.0.0
'
```

**关键**：Knative 默认会把新 Revision **自动接 100% 流量**（如果没配 `traffic`）。要做灰度，必须显式写 `traffic`。

正确做法是一次性提交带 `traffic` 的新配置：

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: recommender
  namespace: shop
spec:
  template:
    metadata:
      name: recommender-v2
      annotations:
        autoscaling.knative.dev/min-scale: "1"
    spec:
      containers:
        - image: registry.example.com/recommender:2.0.0
  traffic:
    - revisionName: recommender-v1
      percent: 95
    - revisionName: recommender-v2
      percent: 5
```

> ⚠️ **注意一个坑**：`percent` 与 `tag` 可以写在同一个 traffic 条目上，但此时 **tag 指向的是「与 percent 对应的那部分流量」，而不是「固定 100% 走 v2」**——线上 5% 分流时，通过该 tag 也只能拿到 5% 的 v2。如果希望预览入口**永远 100% 打到某个 Revision**，就要像案例 3 那样**单独加一条 `percent: 0` + `tag` 的条目**。这两处故意写成不同形式，方便对比。

### 2.5 验证（在 2.4 的基础上补一个预览入口）

```bash
# 额外加一条「预览入口」：不占线上流量，但 tag 100% 指向 v2
kubectl patch ksvc recommender -n shop --type=merge -p '
spec:
  traffic:
    - revisionName: recommender-v1
      percent: 95
    - revisionName: recommender-v2
      percent: 5
    - revisionName: recommender-v2
      percent: 0
      tag: preview
'
```

```bash
# 看流量分配
kubectl get route recommender -n shop -o yaml | grep -A8 'traffic:'

# 看两个 Revision
kubectl get revisions -n shop
# NAME             CONFIG NAME   K8S SERVICE NAME   GENERATION   READY
# recommender-v1   recommender   recommender-v1     1            True
# recommender-v2   recommender   recommender-v2     2            True

# 预览入口：永远 100% 走 v2（不参与线上分流）
curl http://preview-recommender.shop.example.com/recommend?user=123

# 线上入口：按 95/5 分流
curl http://recommender.shop.example.com/recommend?user=123

# 看各自收到的请求数（Prometheus）
#   revision_request_count{revision_name="recommender-v1"}
#   revision_request_count{revision_name="recommender-v2"}
```

### 2.6 逐步放量到 100

```bash
kubectl patch ksvc recommender -n shop --type=merge -p '
spec:
  traffic:
    - revisionName: recommender-v2
      percent: 100
'
```

### 2.7 回滚（1 分钟内）

```bash
kubectl patch ksvc recommender -n shop --type=merge -p '
spec:
  traffic:
    - revisionName: recommender-v1
      percent: 100
'
```

**注意**：回滚只改流量，**不会重新部署**，所以秒级生效；但 `v1` 的 Revision 必须还在（`revisionHistoryLimit` 没被回收）。

### 2.8 关键点

| 点 | 说明 |
| --- | --- |
| 默认行为 | 不写 `traffic` 时，新 Revision 拿 100% |
| 百分比必须合计 100 | 否则 API 报错 |
| Revision 名可用 `metadata.name` 指定 | 否则是自动生成的 `ksvc-00002` |
| 旧 Revision 不会自动缩容到 0 | 需手动或设 `min-scale: 0` |

---

## 案例 3：给内部测试留一个固定入口（Route tag）

### 3.1 业务需求

产品经理要「随时访问最新构建预览版」，但不希望他们影响线上流量。

### 3.2 实现

在 `traffic` 里加 `tag`：

```yaml
spec:
  traffic:
    - revisionName: recommender-v1
      percent: 90
    - revisionName: recommender-v2
      percent: 10
      tag: latest            # 生成固定域名
```

得到的访问地址是：

```
http://latest-recommender.shop.example.com
```

**原理**：`tag` 会在入口层生成一个**独立的 host 路由**，永远指向指定 Revision，且**不参与百分比分流**。

### 3.3 验证

```bash
# 线上入口：按 90/10 分流
curl http://recommender.shop.example.com/recommend

# 预览入口：100% 走 v2
curl http://latest-recommender.shop.example.com/recommend

# 每次发布，把 tag 挪到新 Revision 即可
kubectl patch ksvc recommender -n shop --type=merge -p '
spec:
  traffic:
    - revisionName: recommender-v2
      percent: 100
    - revisionName: recommender-v3
      percent: 0
      tag: latest
'
```

**注意**：`percent: 0` + `tag` 是常用组合 ——「不接线上流量，但可通过 tag 访问」。

---

## 案例 4：夜间批量生成报表（Job + 事件触发）

### 4.1 业务需求

每天凌晨 2 点，给定一个日期范围，生成全站经营报表，跑完就结束。**它不是一个长期服务**。

### 4.2 为什么用 Knative Job Sink

Knative 有 `Job` 类型的 Sink（不是 K8s 的普通 Job），可以让「事件触发一个短命任务」：

- 事件来 → 创建一个 Job → 跑完自动清理；
- 不需要常驻服务；
- 与 Eventing 天然集成。

### 4.3 配置

```yaml
# 1) 定义一个 Knative Job 作为 sink
apiVersion: sink.knative.dev/v1alpha1
kind: Job
metadata:
  name: daily-report
  namespace: ops
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: report
          image: registry.example.com/report-generator:2.1
          args: ["--date", "$(DATE)"]
          env:
            - name: DATE
              value: "today"
      backoffLimit: 2
```

```yaml
# 2) 定时触发（每天 2 点）
apiVersion: sources.knative.dev/v1
kind: PingSource
metadata:
  name: daily-report-trigger
  namespace: ops
spec:
  schedule: "0 2 * * *"
  timezone: "Asia/Shanghai"
  sink:
    ref:
      apiVersion: sink.knative.dev/v1alpha1
      kind: Job
      name: daily-report
```

### 4.4 验证

```bash
# 临时改成 1 分钟一次触发，观察
kubectl patch pingsource daily-report-trigger -n ops --type=merge \
  -p '{"spec":{"schedule":"*/1 * * * *"}}'

kubectl get jobs -n ops -w
# 会看到每天/每分钟生成一个新 Job
```

### 4.5 注意事项

- 定时任务的幂等性很重要（重复触发不能重复出报表）；
- 失败重试策略在 `backoffLimit` 与 Eventing 的重试里要协调，避免指数重试；
- 时区要显式设置（默认 UTC）。

---

## 案例 5：定时爬虫（PingSource + 缩到 0）

### 5.1 业务需求

每小时抓取 3 个竞品网站的价格页，写入数据库。抓取程序本身是个 HTTP 服务。

### 5.2 为什么适合

- 抓取程序平时没人调用 → 缩到 0；
- 事件（定时）驱动 → 一小时后触发时自动拉起；
- 抓取耗时 10–30s，冷启动 3s 可接受。

### 5.3 配置

```yaml
# 1) 爬虫服务（可缩到 0）
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: price-scraper
  namespace: data
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/min-scale: "0"
        autoscaling.knative.dev/max-scale: "5"
    spec:
      timeoutSeconds: 120          # 抓取可能慢
      containerConcurrency: 1      # 每个实例一次只抓一个
      containers:
        - image: registry.example.com/scraper:1.4
          env:
            - name: TARGETS
              value: "https://a.example.com/prices,https://b.example.com/prices"
```

```yaml
# 2) 每小时触发
apiVersion: sources.knative.dev/v1
kind: PingSource
metadata:
  name: hourly-scrape
  namespace: data
spec:
  schedule: "0 * * * *"
  data: '{"mode":"full"}'
  sink:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: price-scraper
```

### 5.4 验证

```bash
# 看 PingSource 状态
kubectl get pingsource hourly-scrape -n data

# 触发一次（临时改成每分钟）
kubectl patch pingsource hourly-scrape -n data --type=merge \
  -p '{"spec":{"schedule":"*/1 * * * *"}}'

# 观察 Pod 被拉起 → 执行 → 缩回 0
kubectl get pods -n data -w
```

### 5.5 注意事项

- **超时**：抓取慢，`timeoutSeconds` 要放大（入口层默认可能是 60s）；
- **PingSource 只负责触发**，不保证业务成功；失败要靠重试 + 监控；
- 抓取频率要遵守对方 robots/协议，别把对方打挂。

---

## 案例 6：上传即处理（Kafka 事件驱动）

### 6.1 业务需求

用户上传视频 → 发一条 Kafka 消息 → 转码服务消费 → 转码完成后再发一条消息 → 通知服务推送。

### 6.2 架构

```
上传服务 ──▶ Kafka(topic: video.uploaded)
                    │
                    ├──▶ KafkaSource ──▶ Broker ──▶ Trigger(type=uploaded) ──▶ 转码服务（缩到 0）
                    │                                              │
                    └──────────────────────────────────────────────┘
                                转码完成 ──▶ 发 video.transcoded
                                                     │
                                                     └──▶ Trigger ──▶ 通知服务
```

### 6.3 配置

```yaml
# 1) Broker（事件中枢）
apiVersion: eventing.knative.dev/v1
kind: Broker
metadata:
  name: media-broker
  namespace: media
---
# 2) Kafka 事件源
apiVersion: sources.knative.dev/v1beta1
kind: KafkaSource
metadata:
  name: video-uploaded
  namespace: media
spec:
  consumerGroup: transcoder
  bootstrapServers:
    - kafka.kafka.svc.cluster.local:9092
  topics:
    - video.uploaded
  sink:
    ref:
      apiVersion: eventing.knative.dev/v1
      kind: Broker
      name: media-broker
---
# 3) 转码服务（缩到 0）
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: transcoder
  namespace: media
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/min-scale: "0"
        autoscaling.knative.dev/max-scale: "20"
        autoscaling.knative.dev/target: "2"     # 转码重，并发要低
    spec:
      timeoutSeconds: 600                       # 转码可能几分钟
      containerConcurrency: 1
      containers:
        - image: registry.example.com/transcoder:3.0
---
# 4) 订阅规则
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: on-video-uploaded
  namespace: media
spec:
  broker: media-broker
  filter:
    attributes:
      type: com.example.video.uploaded
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: transcoder
```

### 6.4 验证

```bash
# 往 Kafka 发一条消息（用 kcat 或临时 Pod）
kubectl run kcat --rm -it --image=edenhill/kcat:1.7.1 -- \
  kcat -b kafka.kafka.svc.cluster.local:9092 -t video.uploaded -P \
  -H "ce-type=com.example.video.uploaded" -l <<< '{"id":"v1","bucket":"videos"}'

# 观察
kubectl get pods -n media -w          # transcoder 被拉起
kubectl logs -n media -l serving.knative.dev/service=transcoder -c user-container
```

### 6.5 注意事项

| 点 | 说明 |
| --- | --- |
| **至少一次投递** | 消息可能重复，转码必须幂等（幂等键 = 视频 ID） |
| **长任务超时** | `timeoutSeconds` 与入口层超时都要放大 |
| **并发要低** | 转码是 CPU/GPU 密集，`target` 设为 1–2 |
| **Kafka offset 语义** | KafkaSource 的消费位点在消息被 sink 接受后才提交（具体语义需核实），失败会重投 |
| **死信队列** | 配置 `deadLetterSink`，避免毒消息无限重试 |

---

## 案例 7：多租户函数托管平台（一个 ksvc 一租户）

### 7.1 业务需求

你做一个 SaaS，客户可以上传「自定义回调函数」，平台负责：按需运行、隔离、配额、计费。

### 7.2 设计

| 需求 | Knative 提供 | 需要平台自己补 |
| --- | --- | --- |
| 按需运行 | 缩到 0 + 事件/请求触发 | — |
| 租户隔离 | 每租户一个 Namespace | RBAC、NetworkPolicy |
| 配额 | — | ResourceQuota + LimitRange |
| 安全 | — | gVisor（见 [`ai-agents/gvisor-kata.md`](./ai-agents/gvisor-kata.md)）+ PSA |
| 计费 | 指标：`revision_request_count`、`revision_request_latencies` | 按调用次数聚合 |
| 代码上线 | Revision 自动版本化 | CI 构建镜像 |

### 7.3 租户模板（示意）

```yaml
# 每个租户一个命名空间 + 配额
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-acme
  labels:
    tenant: acme
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
  namespace: tenant-acme
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    count/jobs.batch: "100"
    serving.knative.dev/services: "50"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-defaults
  namespace: tenant-acme
spec:
  limits:
    - type: Container
      default:        { cpu: 500m, memory: 512Mi }
      defaultRequest: { cpu: 100m, memory: 128Mi }
      max:            { cpu: "4",  memory: 4Gi }
```

用户上传的函数被构建成镜像后，部署为：

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: fn-12345
  namespace: tenant-acme
  labels:
    tenant: acme
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/min-scale: "0"
        autoscaling.knative.dev/max-scale: "10"
    spec:
      containerConcurrency: 5
      timeoutSeconds: 30
      containers:
        - image: registry.example.com/fn/acme/12345:sha256-abc
          securityContext:
            runAsNonRoot: true
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities: { drop: ["ALL"] }
          resources:
            requests: { cpu: 100m, memory: 128Mi }
```

### 7.4 安全加固要点

| 风险 | 措施 |
| --- | --- |
| 用户代码恶意 | gVisor RuntimeClass（`runtimeClassName: gvisor`） |
| 用户代码挖矿 | 限制 CPU/内存 + 网络出口白名单 |
| 用户访问其他租户 | 每租户 Namespace + NetworkPolicy 默认拒绝 |
| 用户读平台密钥 | 每租户独立 ServiceAccount，禁止挂载宿主机路径 |
| 计费争议 | 用 Revision 级指标 + 请求日志 |

### 7.5 注意事项

- **Namespace 数量膨胀**：几千个租户时，控制面压力大，要考虑分集群或 vCluster；
- **冷启动是 SLA 的一部分**：用户函数可能很慢，要评估是否 `min-scale: 1`；
- **构建安全**：用户代码的构建过程本身要隔离（用 Tekton/Buildpacks + 沙箱）；
- **配额单位**：`ResourceQuota` 支持 `serving.knative.dev/services` 这类自定义资源计数。

---

## 总结：案例与能力对照

| 能力 | 对应案例 |
| --- | --- |
| 缩到 0 | 1、5、6 |
| 按请求扩缩 | 1、6 |
| 权重灰度 | 2 |
| 固定预览入口（tag） | 3 |
| 不可变版本与回滚 | 2 |
| 定时事件 | 4、5 |
| 消息事件驱动 | 6 |
| Job 型短任务 | 4 |
| 多租户与配额 | 7 |
| 安全隔离 | 7 |

---

## 附：常用命令速查

```bash
# 服务
kubectl get ksvc -A
kubectl describe ksvc <name> -n <ns>
kubectl get revisions -n <ns>
kubectl get routes -n <ns>

# 触发一次请求并计时（测冷启动）
time curl -s -o /dev/null -w "%{http_code}\n" <URL>

# 实时观察扩缩
kubectl get pods -n <ns> -w

# 事件
kubectl get brokers,triggers,sources -n <ns>
kubectl get pingsource -n <ns>
kubectl get kafkasource -n <ns>

# 指标（Prometheus）
#   revision_request_concurrency{revision_name="..."}
#   revision_request_count{revision_name="..."}
#   autoscaler_desired_pods{revision_name="..."}
```

> ⚠️ 本文基于截止 2026 年中的公开知识整理，当前无法联网核实。`sink.knative.dev` 的 Job 类型、`KafkaSource` 的 API 版本、`PingSource` 的 `timezone` 字段支持情况，请以官方文档为准。
