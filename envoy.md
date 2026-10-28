# Envoy 详解

> 从「它是什么」到「配置怎么写、流量怎么走、线上怎么排障」的完整笔记。
> 阅读建议：先看第一、二章建立心智模型，再对着第三章的 YAML 跑一遍，最后看第六、七章的排障与面试部分。

---

## 目录

- [一、Envoy 是什么](#一envoy-是什么)
- [二、架构与运行模型](#二架构与运行模型)
- [三、核心概念与配置结构](#三核心概念与配置结构)
- [四、动态配置：xDS 协议族](#四动态配置xds-协议族)
- [五、进阶能力](#五进阶能力)
- [六、可观测性](#六可观测性)
- [七、排障与运维](#七排障与运维)
- [八、与 Nginx / HAProxy 的对比](#八与-nginx--haproxy-的对比)
- [九、面试高频问题](#九面试高频问题)
- [附：关键配置字段速查](#附关键配置字段速查)

---

## 一、Envoy 是什么

### 1.1 定义

Envoy 是 Lyft 开源（2016 年）、后捐赠给 CNCF 的高性能 **L4/L7 代理与通信总线**，用 C++11 编写。它最核心的差异点是：

> **配置可以通过网络动态下发（xDS），且热更新不影响存量连接。**

### 1.2 它解决的问题

微服务架构下，每个服务都需要超时、重试、熔断、限流、TLS、灰度、指标、追踪。如果放在应用 SDK 里：

| 问题 | 具体表现 |
| --- | --- |
| 语言碎片化 | Java/Go/Python/Node 各写一套，行为不一致 |
| 升级困难 | 改通信逻辑要改业务代码、重新发版、重新测试 |
| 责任边界模糊 | 业务团队被迫维护网络基础设施代码 |
| 可观测性割裂 | 每个 SDK 打的指标命名、标签都不一样 |

Envoy 的做法是**把通信能力下沉到进程外**（out-of-process）：业务进程只连 `localhost`，所有出入流量都经过 Envoy。

### 1.3 主要能力清单

- **协议**：TCP、HTTP/1.1、HTTP/2、gRPC、WebSocket、HTTP/3（QUIC）。
- **路由**：按 host、path、header、query、method 匹配，支持重写、重定向、镜像。
- **负载均衡**：RR、Least Request、Random、Ring Hash、Maglev、Weighted Least Request，支持 locality 加权。
- **韧性**：超时、重试（含退避与重试预算）、熔断（连接/请求上限）、被动健康检查（outlier detection）、主动健康检查。
- **安全**：TLS 终止、mTLS、SNI、证书热更新（SDS）、JWT 校验、RBAC 过滤器。
- **限流**：本地限流 + 全局限流（配合 ratelimit 服务）。
- **可观测性**：Prometheus 指标、结构化访问日志、分布式追踪（W3C / B3 / Jaeger）、admin 接口。
- **扩展**：HTTP/Network Filter、Lua、Wasm。

### 1.4 一般被用在哪里

1. **服务网格的 sidecar**（Istio 默认数据面）。
2. **API 网关 / Ingress**（Envoy Gateway、Emissary、Gloo Edge）。
3. **边缘代理 / 负载均衡**（替代 Nginx/HAProxy 的部分场景）。
4. **专用代理**：数据库代理（Envoy 的 TCP 模式）、AI 推理网关（新出现的用法）。

---

## 二、架构与运行模型

### 2.1 进程内结构

```
┌──────────────────────────────────────────────────────────┐
│ Envoy Process                                            │
│                                                          │
│  ┌──────────────┐      ┌──────────────────────────────┐  │
│  │ Main Thread  │      │  Worker Threads (N 个)       │  │
│  │              │      │                              │  │
│  │ · 配置管理   │      │  ┌────────┐  ┌────────┐      │  │
│  │ · xDS 客户端 │─────▶│  │Worker 1│  │Worker 2│ ...  │  │
│  │ · Admin API  │ 线程 │  │listen  │  │listen  │      │  │
│  │ · 统计聚合   │ 本地 │  │filters │  │filters │      │  │
│  │              │ 存储 │  │conns   │  │conns   │      │  │
│  └──────────────┘      └──────────────────────────────┘  │
│         │                        ▲                       │
│         ▼                        │                       │
│  ┌──────────────┐        ┌───────┴────────┐              │
│  │ 上游集群状态  │        │ 共享的只读配置  │              │
│  │ (health/LB)  │        │ (TLS slot 指针) │              │
│  └──────────────┘        └────────────────┘              │
└──────────────────────────────────────────────────────────┘
```

要点：

- **每线程一个事件循环（event loop）**，不是每连接一个线程。
- **配置只在 Main 线程变更**，变更后通过「线程本地存储（Thread Local Storage，TLS slot）」把新的只读快照指针发布给所有 Worker。Worker 读配置无锁，更新时只做指针切换。
- **统计信息是线程本地累加**，读取时在 Main 线程聚合，所以高并发下指标开销很小。
- 默认 Worker 数量 = CPU 核数（可用 `--concurrency` 调整）。

### 2.2 为什么这个模型好

- **无锁**：热路径上几乎不争用锁，缓存友好。
- **隔离**：一个 Worker 的事件循环被阻塞（如回调里做慢操作）只影响该线程上的连接，不会全局卡死。
- **热更新平滑**：配置切换是原子指针替换，不需要 reload 进程、不需要清空连接。

### 2.3 与 Nginx 模型的对照

| | Envoy | Nginx |
| --- | --- | --- |
| 进程模型 | 单进程多线程 | master + 多 worker 进程 |
| 配置更新 | 热更新（xDS），不改连接 | `nginx -s reload`：新 worker 起来、旧 worker 优雅退出 |
| 连接保持 | 配置切换不影响 | reload 时旧连接在旧 worker 上，直到关闭 |
| 内存共享 | 线程共享堆，TLS slot 指针 | 进程间靠共享内存区（需要显式设计） |

### 2.4 生命周期与启动流程

```
1. 加载 bootstrap 配置（--config-path 或 --config-yaml）
2. 初始化 Admin 接口（默认 9901）
3. 建立 xDS 连接（若有 dynamic_resources）
4. 接收 LDS/CDS/RDS/EDS/SDS，初始化 Listener 与 Cluster
5. 开始监听并接受连接
6. 周期性地：健康检查、证书轮换、统计 flush、配置 ACK/NACK
```

**bootstrap 配置是唯一必须静态提供的东西**，其它（Listener/Route/Cluster/Endpoint/Secret）都可以动态下发。这也是 Istio 给每个 sidecar 注入的 `envoy-rev0.json`（或 `bootstrap.yaml`）的主要内容：告诉 Envoy「去哪里要配置」。

### 2.5 热重启（Hot Restart）

早期 Envoy 支持 `--restart-epoch` 的热重启：新旧进程共享监听 socket（通过 unix domain socket 传递 fd + 共享内存统计），实现二进制升级不丢连接。现代部署（K8s）中更多依赖**多副本滚动**，热重启使用较少，但概念仍被问到。

---

## 三、核心概念与配置结构

### 3.1 一条请求的路径

```
下游 client
   │
   ▼
Listener (0.0.0.0:8080)
   │
   ├─ Filter Chain 匹配（按 destination IP/port、TLS SNI）
   │
   ├─ Network Filter #1: tls_inspector / proxy_protocol
   ├─ Network Filter #2: http_connection_manager (HCM)
   │       │
   │       ├─ HTTP Filter: cors / jwt_authn / ext_authz / fault / rate_limit
   │       ├─ HTTP Filter: router（最后一个，负责转发）
   │       │
   │       └─ Route 匹配（VirtualHost → Route）
   │              │
   │              ├─ cluster: reviews-v1（权重 90）
   │              └─ cluster: reviews-v2（权重 10）
   │
   ▼
Cluster(round_robin / ring_hash / ...)
   │
   ▼
Endpoint 选择（健康检查、outlier detection 过滤后）
   │
   ▼
上游连接建立（TLS / mTLS / 直连）→ 发送请求
```

### 3.2 Listener 与 Filter Chain

- **Listener** 是「监听地址 + 一组过滤器链」。
- **FilterChainMatch** 决定用哪条链：可以是目标端口、目标 IP、TLS SNI、ALPN 协议。
- 一个 Listener 可以有多个 Filter Chain，但**同一时刻只能匹配一条**（匹配不到会回 `no matching filter chain`，表现为连接被重置）。
- 常见组合：
  - 纯 TCP 代理：`tcp_proxy` filter。
  - HTTP 代理：`http_connection_manager`（HCM）。
  - 带 mTLS 的 sidecar 出站：`tls_inspector` + `original_dst` + HCM。

### 3.3 HTTP Connection Manager（HCM）

HCM 是 Envoy 的 L7 入口，负责：

- 解析 HTTP/1.1 / HTTP/2 / HTTP/3；
- 管理编解码器（codec）、连接复用；
- 执行 HTTP Filter 链；
- 记录访问日志、生成统计、注入 tracing header；
- 处理 `100-continue`、`Expect`、升级（WebSocket）。

HCM 参数举例：

| 字段 | 含义 |
| --- | --- |
| `codec_type` | AUTO / HTTP1 / HTTP2 / HTTP3 |
| `stat_prefix` | 指标前缀，决定 `http.<prefix>.*` 命名 |
| `access_log` | 访问日志输出 |
| `route_config` | 静态路由，或 `rds` 动态订阅 |
| `http_filters` | 过滤器列表，`router` 必须在最后 |
| `use_remote_address` | 是否信任下游真实 IP |
| `normalize_path` | 是否规范化路径（防目录穿越） |
| `merge_slashes` | 是否合并 `//` |
| `stream_idle_timeout` | 流空闲超时 |

### 3.4 Virtual Host 与 Route

- **VirtualHost**：按域名分组的路由集合，`domains: ["reviews", "reviews.shop.svc.cluster.local", "*"]`。
- **Route 匹配顺序**：
  1. `exact`（精确 path）
  2. `prefix`（前缀）
  3. `safe_regex`（正则，性能最差，慎用）
  4. `path_separated_prefix`（按路径段匹配）
- **Header / Query 匹配**：`exact`、`prefix`、`safe_regex`、`present`、`contains`。
- **Route Action**：
  - `route`：转发到某个 cluster，支持 `weighted_clusters`、`hash_policy`（一致性哈希）、`retry_policy`、`timeout`。
  - `redirect`：重定向。
  - `direct_response`：直接返回（用于健康检查、mock）。
- **路由级别能力**：`prefix_rewrite`、`host_rewrite`、`request_mirror_policies`（影子流量）、`cors`、`rate_limits`。

### 3.5 Cluster

Cluster 是「一组上游端点 + 连接策略」。

关键字段：

| 字段 | 说明 |
| --- | --- |
| `type` | `STATIC` / `STRICT_DNS` / `LOGICAL_DNS` / `EDS`（动态端点）/ `ORIGINAL_DST`（保留原目标） |
| `lb_policy` | `ROUND_ROBIN`、`LEAST_REQUEST`、`RANDOM`、`RING_HASH`、`MAGLEV`、`CLUSTER_PROVIDED` |
| `load_assignment` | 静态端点定义（与 EDS 二选一） |
| `health_checks` | 主动健康检查（HTTP/TCP/gRPC） |
| `circuit_breakers` | `max_connections`、`max_pending_requests`、`max_requests`、`max_retries` |
| `outlier_detection` | 被动剔除异常端点 |
| `transport_socket` | TLS / mTLS 配置 |
| `connect_timeout` | 建连超时（默认 5s） |
| `http2_protocol_options` | 启用 HTTP/2 上游（gRPC 必需） |
| `common_lb_config` | 区域感知（locality weighted）、panic 模式 |

> ⚠️ **和 Kubernetes 的 "Cluster" 同名，但完全无关**，见 [3.6](#36-envoy-的-cluster不属于-kubernetes-的-cluster)。

### 3.6 Envoy 的 Cluster，不属于 Kubernetes 的 Cluster

Envoy 文档里的 "Cluster" 是个**高频混淆点**，因为 Kubernetes 里也有 "Cluster"。两者没有任何关系，只是重名。

| | **Envoy Cluster** | **Kubernetes Cluster** |
| --- | --- | --- |
| 是什么 | 一组提供同一服务的上游端点 + 连接策略 | 一整套 K8s 运行环境（控制面 + etcd + 节点） |
| 粒度 | 配置对象，一个 sidecar 里可能有几十上百个 | 基础设施单位，一个代理通常只属于一个 |
| 类比 | 「一条目的地清单」 | 「一整个公司」 |

**一句话**：Envoy 的 Cluster 是「**一组 Pod IP 的逻辑分组**」，Kubernetes 的 Cluster 是「**整个集群**」。

#### 为什么容易混

因为 Istio 给 Envoy cluster 起的名字里，**恰好嵌了 K8s 的域名**：

```
outbound|9080||reviews.shop.svc.cluster.local
                 ^^^^^^ ^^^^^^^
                  │      └─ namespace
                  └─ 这是 K8s 的 Service DNS 后缀
```

- `.svc.cluster.local` 里的 `cluster` 指的是 **K8s 集群的 DNS 域**（可以改）。
- `outbound|9080||reviews.shop.svc.cluster.local` 整体是 **Envoy cluster 的名字**，它只是**借用了 K8s 的域名来命名**，方便阅读。
- Envoy cluster 名字的通用格式是 `<direction>|<port>|<subset>|<host>`，例如 `inbound|9080||`。

**名字里带 cluster，不代表它是一台集群。**

#### "cluster" 一词在同一个技术栈里的五种含义

| 出现位置 | "cluster" 指的是 |
| --- | --- |
| Envoy 配置的 `clusters:` | 一组上游端点 + 负载均衡/连接策略 |
| Istio 生成的 cluster 名字（`outbound\|9080\|\|...`） | 同上，名字借用了 K8s DNS |
| Envoy bootstrap 的 `node.cluster` | **代理所属的逻辑分组**（用于控制面按分组下发配置），又是一个不同的意思 |
| K8s 的 `cluster.local` | K8s 集群的 DNS 域后缀 |
| SPIFFE 身份 `spiffe://cluster.local/ns/<ns>/sa/<sa>` | **信任域（trust domain）的默认名**，可通过 `meshConfig.trustDomain` 修改 |

#### 怎么验证

```bash
# 看这个 sidecar 里到底有多少个"Envoy cluster"
istioctl proxy-config clusters <pod> -n <ns>

# 名字里的 cluster.local 只是 DNS 域名，不是集群对象
istioctl proxy-config clusters <pod> -n <ns> -o json | jq -r '.[].name' | head
```

### 3.7 Endpoint 与健康状态

Endpoint 有几类状态：

- `HEALTHY` / `UNHEALTHY`（主动/被动健康检查）
- `DEGRADED`（稀疏端点降级）
- `DRAINING`（优雅摘除）
- `TIMEOUT`（健康检查超时）

**Panic 模式**：当健康端点比例低于 `healthy_panic_threshold`（默认 50%）时，Envoy 会忽略健康检查结果、把流量发给所有端点。这是「宁可发错也不全挂」的兜底设计，但也会把流量打到异常实例上，排障时要能识别。

### 3.8 最小可运行示例

```yaml
static_resources:
  listeners:
    - name: main
      address:
        socket_address: { address: 0.0.0.0, port_value: 8080 }
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ingress_http
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: backend
                      domains: ["*"]
                      routes:
                        - match: { prefix: "/" }
                          route:
                            cluster: backend
                            timeout: 2s
                            retry_policy:
                              retry_on: "5xx,reset"
                              num_retries: 2
                http_filters:
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
    - name: backend
      type: STRICT_DNS
      lb_policy: LEAST_REQUEST
      load_assignment:
        cluster_name: backend
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address: { address: backend, port_value: 80 }
      circuit_breakers:
        thresholds:
          - priority: DEFAULT
            max_connections: 100
            max_pending_requests: 50
            max_requests: 200

admin:
  address:
    socket_address: { address: 0.0.0.0, port_value: 9901 }
```

> 注意 `@type` 字段：Envoy v3 API 使用 protobuf Any，每个过滤器都要写类型 URL，这也是手写配置容易出错的地方。

---

## 四、动态配置：xDS 协议族

### 4.1 什么是 xDS

xDS = 「x Discovery Service」的统称，是一组 **gRPC 流式协议**，让 Envoy 从控制面订阅自己需要的配置，而不是启动时写死。

| 协议 | 管什么 | 依赖关系 |
| --- | --- | --- |
| **LDS** | Listener | 需要 cluster 已存在（HCM 引用） |
| **RDS** | RouteConfiguration | 需要 cluster 已存在 |
| **CDS** | Cluster | - |
| **EDS** | Cluster 内的 Endpoint | 需要对应 cluster 存在 |
| **SDS** | TLS 证书与密钥 | 被 Listener/Cluster 的 transport_socket 引用 |
| **VHDS** | VirtualHost | 按需拉取，减少 RDS 体积 |
| **ADS** | Aggregated Discovery Service | 把以上合并到一个流，保证顺序一致性 |

**为什么需要 ADS**：如果 LDS 先于 CDS 到达，Listener 引用的 Cluster 还不存在，Envoy 会 NACK。ADS 通过「一个双向流 + 版本校验」保证配置的整体一致性，是 Istio 和主流控制面的默认方式。

### 4.2 版本与变体

- `State of the World`（SotW）：每次全量下发某个资源列表，简单但大。
- **Delta xDS**：增量下发（新增/更新/删除单个资源），适合大规模（如百万级 endpoint）。
- **TLS / mTLS**：xDS 本身也可以走 TLS，控制面与数据面之间要做认证。
- **xDS 客户端也可以不是 Envoy**（gRPC 的 xDS resolver、Consul 等）。

### 4.3 ACK / NACK 机制

```
控制面                                Envoy
  │  DiscoveryResponse(version=3)  ──▶
  │                                  ├─ 校验 schema / 引用
  │                                  ├─ 成功后应用，回 ACK
  │  ◀── DiscoveryRequest(ACK, v3) ──┤
  │                                  │
  │  DiscoveryResponse(version=4) ──▶
  │                                  ├─ 校验失败，回 NACK
  │  ◀── DiscoveryRequest(NACK, v3) ─┤  (保留 v3，不生效 v4)
```

- NACK 会让控制面知道哪个版本有问题，但**Envoy 会继续用旧配置**，所以「配置错误不一定表现为故障」，而是控制面指标里 `nack` 数量上升。
- 排查要同时看：控制面日志 + `istioctl proxy-status`（SYNCED / STALE / NOT SENT）+ Envoy 的 `config_dump`。

### 4.4 bootstrap 中的 xDS 配置示例

```yaml
node:
  id: sidecar~10.0.0.5~reviews-v1-abc~shop.svc.cluster.local
  cluster: reviews.shop

dynamic_resources:
  ads_config:
    api_type: GRPC
    grpc_services:
      - envoy_grpc: { cluster_name: xds_cluster }
  cds_config: { ads: {} }
  lds_config: { ads: {} }

static_resources:
  clusters:
    - name: xds_cluster
      type: STRICT_DNS
      load_assignment:
        cluster_name: xds_cluster
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address: { address: istiod-plus, port_value: 15012 }
      http2_protocol_options: {}
```

> `node.id` 是 Envoy 的身份标识，控制面用它来决定「这个代理应该看到哪些配置」。Istio 会把 namespace、ServiceAccount、Pod 名编码进去，这也是多租户与「Sidecar 资源」能生效的基础。

---

## 五、进阶能力

### 5.1 负载均衡策略怎么选

| 场景 | 推荐策略 | 原因 |
| --- | --- | --- |
| 请求耗时均匀 | Round Robin | 简单、均匀 |
| 请求耗时差异大 | Least Request | 按在途请求数分配，避免慢节点堆积 |
| 需要会话粘性 / 缓存命中 | Ring Hash / Maglev | 按 key 哈希，后端增减时影响小 |
| 大集群 + gRPC 长连接 | Maglev | 哈希表 O(1)，一致性更好 |
| 跨区域 | Locality Weighted LB | 优先本地，异常时溢出到其他区域 |

**一致性哈希的坑**：默认的 Ring Hash 使用固定 ring size，节点变化时会有部分抖动；`Maglev` 在大规模下更稳定。gRPC 场景若客户端不做 LB，Envoy 必须开启 `http2_protocol_options` 并注意长连接粘连。

### 5.2 熔断与 outlier detection

Envoy 的「熔断」是**并发限制**，不是「错误率熔断」：

```yaml
circuit_breakers:
  thresholds:
    - priority: DEFAULT
      max_connections: 100
      max_pending_requests: 50
      max_requests: 200
      max_retries: 3
```

- 超过限制时返回 `503`，统计里看到 `upstream_rq_pending_overflow` 或 `upstream_rq_pending_active`。
- **Outlier Detection** 才是「被动健康检查」：连续 5xx 次数超过阈值 → 剔除端点 `base_ejection_time` → 到时间后放回；如果放回后仍失败，剔除时间指数增长。
- 主动健康检查（`health_checks`）更适合明确的 liveness 语义，但要注意它本身会消耗资源（大规模时降低频率或改用被动）。

### 5.3 重试与超时

- `retry_policy`：`retry_on`（`5xx`、`reset`、`connect-failure`、`retriable-status-codes`）、`num_retries`、`per_try_timeout`、`retriable_status_codes`、`retry_back_off`。
- **重试放大问题**：每层都加重试时，实际请求量会指数放大。Envoy 支持 `retry_budget`（重试预算）来限制全局重试比例。
- 超时层级：`Route.timeout`（整体）> `per_try_timeout`（单次）；`Cluster.connect_timeout` 建连阶段。
- 常见坑：`timeout: 0s` 表示禁用超时（无限），排查时发现「请求挂住」多半是这个原因。

### 5.4 TLS 与 mTLS

- `transport_socket` 配 `UpstreamTlsContext` / `DownstreamTlsContext`。
- 支持 `SDS` 动态证书，证书轮换不重启。
- mTLS 校验可使用：
  - `require_client_certificate: true`（强制客户端证书）。
  - `validation_context` + `trusted_ca`（信任链）。
  - `verify_certificate_hash` / `verify_spki_hash`（更严格的校验）。
- 双向证明确认对方身份后，可用 `ext_authz`/`rbac` 根据证书的 SAN 做授权。

### 5.5 限流

- **本地限流**：`local_ratelimit`（令牌桶），单实例生效。
- **全局限流**：`envoy.filters.http.ratelimit` + 外部 `ratelimit` 服务（gRPC），按 key（如 header 中的用户 ID）在集群维度限流。
- `rate_limits` 中定义的 `actions`（如 `remote_address`、`request_headers`、`generic_key`）决定了限流维度。

### 5.6 扩展方式

| 方式 | 说明 | 适用 |
| --- | --- | --- |
| 官方过滤器 | 覆盖 90% 需求 | 优先 |
| Lua | 轻量脚本，性能一般 | 小逻辑 |
| **Wasm** | 多语言（Rust/C++/Go/AssemblyScript），沙箱、可热加载 | 自定义业务逻辑 |
| C++ 过滤器 | 性能最好，需重编译 | 极限性能 |
| ext_authz / ext_proc | 外部服务处理 | 复杂策略 |

Istio 的 `WasmPlugin` 就是基于 Envoy 的 Wasm 扩展能力。

### 5.7 Envoy Gateway 与功能定位

Envoy 本身可以被当作「数据面组件」，但用户还需要一个「配置入口」。K8s 生态里常见：

- **Envoy Gateway**：CNCF 项目，实现 Gateway API 的控制器，底层用 Envoy 作为数据面（xDS 由 EG 下发）。
- **Emissary-ingress**：以注解/CRD 配置 Envoy。
- **Istio**：用 Istio CRD 配置 Envoy。

理解这一点：**Envoy 是引擎，Gateway/Istio 是方向盘。**

---

## 六、可观测性

### 6.1 指标（Prometheus）

默认暴露在 `/stats/prometheus` 或 admin 的 `:9901/stats/prometheus`，也可用 `stats_sinks` 推送到 statsd/OTel。

最常看的：

| 指标 | 含义 |
| --- | --- |
| `downstream_cx_total` / `downstream_cx_active` | 下游连接数 |
| `upstream_cx_total` / `upstream_cx_active` | 上游连接数 |
| `upstream_rq_total` / `upstream_rq_time` | 上游请求数与延迟 |
| `upstream_rq_5xx` / `upstream_rq_503` | 错误 |
| `upstream_rq_pending_overflow` | 溢出（熔断触发） |
| `upstream_rq_timeout` | 超时 |
| `upstream_cx_connect_fail` | 建连失败 |
| `cluster.<name>.membership_healthy` | 健康端点数量 |

**排查 503 的经典流程**：看 `upstream_rq_503` → 看 `membership_healthy` 是否为 0 → `config_dump` 看端点是否下发 → 看是否 `pending_overflow`（熔断）或 `outlier ejection`（被剔除）。

### 6.2 访问日志

```yaml
access_log:
  - name: envoy.access_loggers.stdout
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
      log_format:
        json_format:
          start_time: "%START_TIME%"
          method: "%REQ(:METHOD)%"
          path: "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%"
          status: "%RESPONSE_CODE%"
          duration: "%DURATION%"
          upstream: "%UPSTREAM_HOST%"
          trace_id: "%REQ(X-B3-TRACEID)%"
```

**关键字段**：
- `%UPSTREAM_HOST%`：实际转发的后端 IP:Port（排查「到底打到谁」）。
- `%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%`：重写前的原始路径。
- `%DURATION%`、`%RESPONSE_FLAGS%`（如 `UF`=upstream failure、`NR`=no route、`UH`=no healthy upstream）。

### 6.3 分布式追踪

- Envoy 可以**透传** trace header（W3C `traceparent`、B3），也可以**生成 span**（需 `tracing` 配置）。
- 通常由 sidecar 生成入站/出站 span，应用只需透传 header（或由 SDK 生成）。
- 采样率配置在 `tracing.random_sampling` 或 `client_sampling`。

---

## 七、排障与运维

### 7.1 Admin 接口（:9901）

| 路径 | 用途 |
| --- | --- |
| `/config_dump` | 当前生效的全部配置（listener/route/cluster/endpoint） |
| `/clusters?format=json` | 集群与端点状态（健康、统计） |
| `/listeners` | Listener 列表 |
| `/stats` | 全部统计 |
| `/stats/prometheus` | Prometheus 格式 |
| `/server_info` | 版本、uptime、状态 |
| `/ready` | 是否 ready |
| `/logging` | 动态调整日志级别 |
| `/hot_restart_version` | 热重启兼容性 |
| `/quitquitquit` | 退出（慎用） |

在 sidecar 里：

```bash
kubectl exec <pod> -c istio-proxy -- curl -s localhost:15000/config_dump | jq '.configs[].@type'
kubectl exec <pod> -c istio-proxy -- curl -s "localhost:15000/clusters?format=json" | jq '.cluster_statuses[] | {name, hosts: [.host_statuses[].address.socket_address.address]}'
kubectl exec <pod> -c istio-proxy -- curl -s localhost:15000/stats | grep 'upstream_rq_'
```

### 7.2 典型故障对照表

| 现象 | 可能原因 | 排查点 |
| --- | --- | --- |
| `503 UF` | 上游连接失败（端口错/未监听/TLS 不匹配） | `%RESPONSE_FLAGS%`、`upstream_cx_connect_fail`、后端是否在监听 |
| `503 UH` | 没有健康上游 | `membership_healthy`、outlier detection 是否把端点全剔了 |
| `503 NR` | 没有匹配路由 | `config_dump` 的 route config、域名是否匹配 |
| `503 NC` | 没有匹配 Cluster | 路由指向的 cluster 不存在（xDS 顺序问题） |
| `503 UO` | 溢出 / 熔断 | `circuit_breakers`、`upstream_rq_pending_overflow` |
| 请求挂住不返回 | `timeout: 0s` 或上游不响应 | route timeout、后端日志 |
| 延迟抖动 | Worker 阻塞、CPU 限流、DNS 解析慢 | `envoy_server_concurrency`、`kubectl top` |
| 配置不生效 | NACK、版本未推进、控制面未下发 | `config_dump` + 控制面日志 |
| 连接被重置 | filter chain 不匹配、TLS 握手失败 | `listener_manager` 日志、SNI/ALPN |

### 7.3 日志级别动态调整

```bash
# 把连接池日志调到 debug
kubectl exec <pod> -c istio-proxy -- curl -X POST "localhost:15000/logging?connection=debug"
# 恢复
kubectl exec <pod> -c istio-proxy -- curl -X POST "localhost:15000/logging?connection=info"
```

线上排查时**只调特定组件**（`connection`、`router`、`upstream`），不要全局 debug（日志量与性能开销都很大）。

### 7.4 性能与容量

- 每个连接的内存与 buffer 开销：默认 `per_connection_buffer_limit_bytes: 32KB`。
- CPU：主要消耗在 TLS 握手、HTTP/2 编解码、正则匹配、Lua/Wasm。
- 大流量场景：调大 `--concurrency`、限制日志、减少 `safe_regex`、开启 HTTP/2 多路复用。
- 已知瓶颈：`/config_dump` 在配置巨大时很重，生产慎用；可改用 `proxy-config` 分项查询。

---

## 八、与 Nginx / HAProxy 的对比

| 维度 | Envoy | Nginx | HAProxy |
| --- | --- | --- | --- |
| 定位 | 服务网格通用数据面 | Web 服务器 + 反向代理 | 高性能 TCP/HTTP 负载均衡 |
| 动态配置 | xDS 原生 | 商业版 API；开源需 reload | Runtime API（部分）；配置 reload |
| L7 协议 | HTTP/1.1、HTTP/2、gRPC、HTTP/3 | HTTP/1.1、HTTP/2（较晚）、HTTP/3（新版本） | HTTP/1.1、HTTP/2 |
| 可观测性 | 内建指标/日志/追踪，结构统一 | 需模块 | 内建统计与日志 |
| 扩展 | Filter、Lua、Wasm | 模块（C）、Lua | Lua、SPOE |
| 资源隔离 | 线程 + 事件循环，热更新不抖 | 进程模型，reload 有抖动 | 进程/线程模型 |
| 学习曲线 | 陡（protobuf、filter 链） | 平缓 | 中等 |
| 典型场景 | 网格 sidecar、网关 | 边缘 Web 服务、Ingress | 四层负载均衡 |

一句话总结：**Nginx 是「一台好用的 Web 服务器」，Envoy 是「一个可被程序控制的流量引擎」。**

---

## 九、面试高频问题

**Q1：Envoy 为什么适合做 sidecar？**
进程外、语言无关、配置可动态下发、热更新不丢连接、内建可观测性；单进程多线程，资源开销可控（闲置时约 30–50MB）。

**Q2：Envoy 的线程模型？**
Main 线程管配置与 xDS，Worker 线程各自跑事件循环处理连接；配置通过 TLS slot 无锁下发。

**Q3：xDS 有哪几个？为什么需要 ADS？**
LDS/RDS/CDS/EDS/SDS；ADS 把多个资源合并为一个流，保证 LDS 引用 CDS 时的顺序一致性，避免 NACK。

**Q4：Envoy 配置错误会怎样？**
校验失败回 NACK，**保留旧配置继续运行**，所以可能「配置没生效但业务正常」；要看控制面指标和 `config_dump`。

**Q5：熔断和 outlier detection 的区别？**
熔断是「并发/连接数上限」的主动限制（到阈值返回 503）；outlier detection 是根据错误率/连续错误被动剔除端点。前者防过载，后者防坏实例。

**Q6：Panic 模式是什么？**
当健康端点比例低于阈值（默认 50%）时，Envoy 忽略健康检查、把流量发给所有端点，避免服务完全不可用；代价是可能打到故障实例。

**Q7：如何排查 503？**
看 `RESPONSE_FLAGS`（UF/UH/NR/NC/UO）+ `clusters` 中的 `membership_healthy` + `config_dump` 看路由/端点下发。

**Q8：Envoy 怎么做长连接负载均衡？**
gRPC/HTTP2 长连接下，LB 在连接建立时决定后端，后续请求都在同一连接上；可用 `max_requests_per_connection`（定期断开重连）或客户端侧 LB（ring hash/maglev）缓解倾斜。

**Q9：Envoy 和 Nginx 最大的区别？**
配置模型：Envoy 是 protobuf + xDS 动态配置，Nginx 是静态文件 + reload。

**Q10：Envoy 如何做 TLS 轮换？**
SDS 动态下发证书，或文件监听自动 reload；不需要重启进程，也不用断开现有连接（新连接用新证书）。

---

## 附：关键配置字段速查

| 层级 | 字段 | 作用 |
| --- | --- | --- |
| Listener | `address` / `filter_chains` | 监听地址与过滤器链 |
| Listener | `filter_chain_match` | 按端口/SNI/ALPN 选择链 |
| HCM | `stat_prefix` | 指标前缀 |
| HCM | `route_config` / `rds` | 静态或动态路由 |
| HCM | `http_filters` | HTTP 过滤器链（`router` 最后） |
| HCM | `use_remote_address` | 使用下游真实 IP |
| Route | `match` | path/header/query 匹配 |
| Route | `route.weighted_clusters` | 权重分流（灰度） |
| Route | `route.retry_policy` / `timeout` | 重试与超时 |
| Route | `request_mirror_policies` | 影子流量 |
| Cluster | `type` / `lb_policy` | 发现方式与负载均衡策略 |
| Cluster | `circuit_breakers` | 并发/连接上限 |
| Cluster | `outlier_detection` | 被动健康检查 |
| Cluster | `transport_socket` | TLS/mTLS |
| Cluster | `connect_timeout` | 建连超时 |
| Cluster | `http2_protocol_options` | 上游 HTTP/2（gRPC） |
| Endpoint | `health_status` | HEALTHY/UNHEALTHY/DEGRADED |
| Admin | `/config_dump`、`/clusters` | 排障入口 |
