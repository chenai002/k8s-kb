# Istio 详解

> 从「为什么需要服务网格」到「能独立设计、排障、升级」的完整笔记。
> 建议与同目录的 [`envoy.md`](./envoy.md) 配合阅读：Istio 是控制面，Envoy 是数据面。

---

## 目录

- [一、Istio 是什么](#一istio-是什么)
- [二、为什么需要服务网格](#二为什么需要服务网格)
- [三、架构演进与组件](#三架构演进与组件)
- [四、流量劫持：sidecar 如何工作](#四流量劫持sidecar-如何工作)
- [五、核心资源配置详解](#五核心资源配置详解)
- [六、安全：身份与 mTLS](#六安全身份与-mtls)
- [七、可观测性](#七可观测性)
- [八、Ambient 模式](#八ambient-模式)
- [九、一次请求的完整旅程](#九一次请求的完整旅程)
- [十、安装、升级与运维](#十安装升级与运维)
- [十一、排障手册](#十一排障手册)
- [十二、面试高频问题](#十二面试高频问题)
- [附：资源对象速查](#附资源对象速查)

---

## 一、Istio 是什么

### 1.1 定义

Istio 是一个开源的 **服务网格（Service Mesh）** 平台，用统一的方式为微服务提供：

- **流量管理**：路由、灰度、超时、重试、熔断、镜像、故障注入；
- **安全**：自动 mTLS、身份认证、细粒度授权；
- **可观测性**：统一指标、访问日志、分布式追踪；
- **策略**：限流、配额、外部授权。

它由控制面 `istiod` + 数据面 `Envoy` 组成，**声明式配置写在 Kubernetes CRD 里**，由控制面翻译成 Envoy 的 xDS 下发。

### 1.2 核心理念

1. **把通信能力从应用代码中剥离**：业务只关心业务，网络策略由平台声明。
2. **配置面向「服务」而不是「机器」**：`host: reviews`，而不是写一堆 IP。
3. **控制面与数据面分离**：配置变更不重启业务，流量路径与配置分发解耦。
4. **身份是一等公民**：每个工作负载都有 SPIFFE 身份，用于加密与授权。

---

## 二、为什么需要服务网格

### 2.1 微服务通信的常见痛点

| 痛点 | 表现为 |
| --- | --- |
| 韧性代码重复 | 每个服务各自实现重试、超时、熔断，参数不一致 |
| 多语言不一致 | Java/Go/Python 的 SDK 行为差异 |
| 发布策略分散 | 灰度靠改代码或改 Nginx，缺乏统一口径 |
| 安全边界模糊 | 服务间明文通信，身份靠 IP 白名单 |
| 观测割裂 | 每个团队自建指标与日志，无法拼出全局拓扑 |
| 依赖治理缺失 | 无法统一回答「谁在调用这个服务」「哪个版本出错多」 |

### 2.2 三种解决方案对比

| 方案 | 做法 | 优点 | 缺点 |
| --- | --- | --- | --- |
| SDK 库（如 Spring Cloud / gRPC 拦截器） | 通信逻辑嵌入应用 | 性能好、无额外进程 | 语言绑定、升级要发版、难以统一治理 |
| 网关集中（Ingress + 出口代理） | 东西向仍直连 | 简单 | 无法覆盖东西向 |
| **服务网格（sidecar）** | 每个 Pod 挂代理，控制面统一配置 | 语言无关、策略统一、可观测 | 资源开销、复杂度、排查链路变长 |

### 2.3 服务网格的「代价」也要清楚

- 每个 Pod 多一个容器（内存 50–100MB，CPU 少量）。
- 请求多一跳（localhost → sidecar），延迟增加约 1–3ms。
- 排障链路变长：应用日志 → sidecar 日志 → 控制面配置。
- 升级 sidecar 需要重启业务 Pod（sidecar 版本与业务 Pod 生命周期绑定）。
- iptables 劫持与 NetworkPolicy、hostNetwork 有交互问题。

这些正是 **Ambient 模式** 要解决的问题。

---

## 三、架构演进与组件

### 3.1 历史版本对照

| 版本 | 控制面构成 | 说明 |
| --- | --- | --- |
| 1.0 – 1.4 | Pilot、Citadel、Galley、Mixer | Mixer 负责策略与遥测，性能瓶颈明显；Galley 负责配置校验 |
| 1.5 | 合并为 **istiod** | Pilot + Citadel + Galley 合一；Mixer 逐渐移除 |
| 1.7+ | istiod + Envoy + 网关 | 默认使用 Wasm 扩展替代 Mixer 适配器 |
| 1.17+ | **Ambient 模式 alpha** | ztunnel（L4）+ waypoint（L7） |
| 目前 | istiod + sidecar / ambient | Gateway API 成为推荐入口方案 |

**面试要点**：Mixer 被移除的原因是「每个请求都要 RPC 调用 Mixer 做策略检查」，延迟与扩展性都不可接受；新方案把策略能力下放到 Envoy filter（ext_authz、Wasm）。

### 3.2 当前架构

```
┌────────────────────────────────────────────────────────────┐
│                       控制面 istiod                         │
│                                                            │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Pilot       │  │ Citadel      │  │ 配置校验/推送     │  │
│  │ · 监听 K8s  │  │ · 证书签发   │  │ · 合并多资源      │  │
│  │ · 生成 xDS  │  │ · 轮换       │  │ · 按需下发(VHDS) │  │
│  └─────────────┘  └──────────────┘  └──────────────────┘  │
│         │                    │                             │
│         │ xDS (gRPC, 明文/    │ SDS (证书)                  │
│         │ mTLS, 双向流)       │                             │
└─────────┼────────────────────┼─────────────────────────────┘
          ▼                    ▼
   ┌─────────────────────────────────────┐
   │  数据面                              │
   │  · sidecar（Envoy，每 Pod 一个）      │
   │  · 或 Ambient：ztunnel + waypoint     │
   │  · 入口/出口网关（Envoy Deployment）   │
   └─────────────────────────────────────┘
```

### 3.3 istiod 内部职责

| 模块 | 职责 |
| --- | --- |
| **Pilot** | 监听 Service/EndpointSlice/CRD，转换为 CDS/EDS/RDS/LDS 下发 |
| **Citadel** | 为 ServiceAccount 签发 X.509 证书（SDS 下发），实现 mTLS |
| **Config Store** | 维护 Istio CRD 与 K8s 资源的内部快照，处理最终一致性 |
| **Webhook** | sidecar 注入（Mutating）、配置校验（Validating） |
| **CA 后端** | 默认自签（`cacerts` Secret 或自动生成），可对接 Vault/自建 CA |

**istiod 高可用**：多副本 + leader election（故障时接管），配置缓存；`PILOT_CERT_PROVIDER`、`PILOT_TRACE_SAMPLING` 等环境变量可调。

### 3.4 控制面故障的影响

| 场景 | 影响 |
| --- | --- |
| istiod 全部挂掉 | 已有 sidecar 继续用缓存配置转发；新 Pod 无法注入、配置变更不生效、证书无法轮换（已有证书到期后会失败） |
| istiod 慢 | 配置下发延迟，`proxy-status` 显示 STALE |
| 证书签发失败 | 新 Pod 的 sidecar 无法建立 mTLS，报 `certificate not available` |

---

## 四、流量劫持：sidecar 如何工作

### 4.1 注入后 Pod 的结构

```
Pod
├── istio-proxy (Envoy)          # sidecar
│   ├── :15001  出站劫持
│   ├── :15006  入站劫持
│   ├── :15000  admin
│   ├── :15020  合并指标 /healthz
│   ├── :15021  就绪探针
│   └── :15090  Prometheus 指标
├── istio-init (已退出)           # 负责写 iptables 规则
└── app container                 # 业务，监听 8080
```

### 4.2 iptables 劫持原理

`istio-init` 在 Pod 的 network namespace 里写入规则（简化）：

```bash
# 出站：非 15001 用户的流量重定向到 15001
iptables -t nat -A ISTIO_OUTPUT -m owner --uid-owner 1337 -j RETURN   # Envoy 自己跳过
iptables -t nat -A ISTIO_OUTPUT -j REDIRECT --to-port 15001

# 入站：所有入站流量重定向到 15006
iptables -t nat -A ISTIO_INBOUND -p tcp -j REDIRECT --to-port 15006
```

- Envoy 以 UID 1337 运行，避免自我循环。
- `ISTIO_IN_REDIRECT` 与 `ISTIO_OUTPUT` 链是 Istio 特有的组织方式，可用 `istio-iptables` 参数调整。

**副作用**：
- **hostNetwork Pod 无法被劫持**（规则写在 Pod ns，但流量走宿主 ns）。
- 与 NetworkPolicy 的交互：策略作用于原始目标，但流量实际被 sidecar 转发，需注意 CNI 的视角。
- **出站 DNS 也会被劫持**（1.8+ 的 DNS capture），可以让 sidecar 做 DNS 缓存与自动服务发现。

### 4.3 流量走向

**出站**：应用 → `reviews:9080`（DNS 返回 ClusterIP）→ iptables 重定向到 `15001` → Envoy 根据目标 IP 找到对应 Cluster（Istio 会生成 `reviews.shop.svc.cluster.local` 的 Cluster）→ 按 LB 选择 Pod IP → 建连（mTLS）→ 对端 `15006`。

**入站**：外部 → 目标 Pod IP:9080 → `15006` → Envoy 执行 AuthorizationPolicy、mTLS 校验 → 转发到 `127.0.0.1:8080`。

**关键点**：**Service 的 ClusterIP 不参与最终的负载均衡**，Envoy 拿到的是 Endpoint 列表，自己选实例。这就是为什么「kube-proxy 的会话保持」在网格里失效。

### 4.4 注入方式

```bash
# 1. 命名空间级
kubectl label namespace shop istio-injection=enabled

# 2. 或按 revision（多版本控制面）
kubectl label namespace shop istio.io/rev=1-20-0

# 3. Pod 级覆盖
annotations:
  sidecar.istio.io/inject: "false"     # 关闭
  sidecar.istio.io/proxyCPU: "100m"    # 调整 sidecar 资源
  proxy.istio.io/config: |
    holdApplicationUntilProxyStarts: true
```

**注意**：命名空间打 label 只影响之后创建的 Pod，已有 Pod 不会自动注入。

### 4.5 启动顺序与「应用先于代理就绪」问题

默认情况下容器并行启动，应用可能在 Envoy 就绪前就开始发请求，导致启动失败。解决方案：

- `holdApplicationUntilProxyStarts: true`：让应用等待 sidecar ready（会有启动延迟）。
- 应用层加重试。
- 原生 Sidecar（K8s 1.29+）可以让 istiod 以 init container 形式注入，保证顺序。

---

## 五、核心资源配置详解

### 5.1 资源全景

| 类别 | 资源 | 一句话 |
| --- | --- | --- |
| 流量 | **VirtualService** | 怎么路由（匹配 + 动作） |
| 流量 | **DestinationRule** | 到了之后怎么连（子集、LB、连接池、mTLS） |
| 流量 | **Gateway** | 入口/出口监听定义 |
| 流量 | **ServiceEntry** | 把外部服务纳入网格 |
| 流量 | **Sidecar** | 限制工作负载能「看到」的服务范围 |
| 流量 | **WorkloadEntry** | 把非 K8s 工作负载（VM）纳入网格 |
| 安全 | **PeerAuthentication** | 工作负载间 mTLS 模式 |
| 安全 | **RequestAuthentication** | 校验 JWT（OIDC） |
| 安全 | **AuthorizationPolicy** | 允许/拒绝（L4/L7） |
| 可观测 | **Telemetry** | 指标/日志/追踪配置 |
| 扩展 | **EnvoyFilter** | 直接打补丁到底层 Envoy（慎用） |
| 扩展 | **WasmPlugin** | 加载 Wasm 过滤器 |
| 运维 | **IstioOperator** | 安装配置（或 Helm values） |

### 5.2 VirtualService

**作用**：定义「请求如何被匹配、如何被处理」。

结构：
```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
  namespace: shop
spec:
  hosts:                       # 一组域名（可以是 Service 名、外部域名）
    - reviews
    - reviews.shop.svc.cluster.local
  gateways:                    # 留空表示只作用于网格内部
    - mesh
  http:
    - name: canary
      match:
        - headers: { x-canary: { exact: "true" } }
      route:
        - destination: { host: reviews, subset: v2 }
    - route:
        - destination: { host: reviews, subset: v1, port: { number: 9080 } }
          weight: 90
        - destination: { host: reviews, subset: v2 }
          weight: 10
      timeout: 2s
      retries:
        attempts: 3
        perTryTimeout: 500ms
        retryOn: 5xx,reset,connect-failure
```

**匹配维度**：
- `uri`（exact/prefix/regex）、`method`、`headers`、`queryParams`、`port`、`sourceLabels`、`gateways`。
- 匹配顺序：从上到下，**第一个匹配的规则生效**。

**动作**：
- `route`（转发，支持权重、`destination.port`、`headers` 操作）。
- `redirect`（301/302/303/307/308）。
- `rewrite`（`uri`、`authority`）。
- `fault`（注入延迟/中止，用于混沌测试）。
- `mirror` + `mirrorPercentage`（影子流量）。
- `corsPolicy`、`retries`、`timeout`、`headers`（request/response 增删）。

**常见坑**：
- `hosts` 写错（写成 ClusterIP 或不带 namespace 的短名，在跨 namespace 时匹配不到）。
- `timeout` 与 `retries` 冲突：`timeout` 是整体超时，`perTryTimeout × attempts` 应小于它。
- 灰度权重只作用于 **已经命中 VS 的流量**，若上游走了 IP 直连则无效。

### 5.3 DestinationRule

**作用**：定义「到达目标服务后的策略」。

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
  namespace: shop
spec:
  host: reviews.shop.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST
    connectionPool:
      tcp: { maxConnections: 100, connectTimeout: 1s }
      http: { http1MaxPendingRequests: 50, http2MaxRequests: 200 }
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 60s
      maxEjectionPercent: 50
    tls:
      mode: ISTIO_MUTUAL
  subsets:
    - name: v1
      labels: { version: v1 }
    - name: v2
      labels: { version: v2, track: canary }
```

**要点**：
- `subsets` 依赖 **Pod 的 label**（通常 `version`），需要 Deployment 模板里打上对应 label。
- `host` 必须与 VirtualService 的 destination host 对应（推荐用 FQDN）。
- `trafficPolicy` 可在 `spec` 级别（全局）与 `subsets[].trafficPolicy`（局部）分别覆盖。
- `tls.mode`：`DISABLE`、`SIMPLE`、`MUTUAL`、`ISTIO_MUTUAL`（网格内自动用 istiod 签发的证书）。
- 一致性哈希：`loadBalancer.consistentHash`（`httpHeaderName`、`httpCookie`、`useSourceIp`、`ringHash`/`maglev`）。

**VirtualService 与 DestinationRule 的关系**：
VS 引用 `destination.host` + `subset`，DR 定义 `host` 与 `subsets`。**没有 DR 时，VS 只能按 Service 级别路由，无法使用 subset。**

### 5.4 Gateway

两种 API：

**A. Istio 原生 Gateway（`networking.istio.io`）**

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway
  namespace: shop
spec:
  selector:
    istio: ingressgateway        # 选中 ingressgateway 的 Deployment
  servers:
    - port: { number: 80, name: http, protocol: HTTP }
      hosts: ["bookinfo.example.com"]
    - port: { number: 443, name: https, protocol: HTTPS }
      tls:
        mode: SIMPLE
        credentialName: bookinfo-cert    # 引用 Secret
      hosts: ["bookinfo.example.com"]
```

然后由 VirtualService 的 `gateways` 字段绑定。**注意 `servers.hosts` 与 VS 的 `hosts` 要一致（或使用 `*`）。**

**B. Kubernetes Gateway API（推荐）**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata: { name: bookinfo-gw, namespace: shop }
spec:
  gatewayClassName: istio
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes: { namespaces: { from: Same } }
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: bookinfo, namespace: shop }
spec:
  parentRefs: [{ name: bookinfo-gw }]
  hostnames: ["bookinfo.example.com"]
  rules:
    - matches: [{ path: { type: PathPrefix, value: /productpage } }]
      backendRefs: [{ name: productpage, port: 9080 }]
```

**Gateway API 的优势**：角色分离（基础设施/应用）、跨 namespace、标准化字段、更丰富的匹配。

### 5.5 ServiceEntry

把网格外的依赖（外部 API、数据库、VM 上的服务）纳入网格，让 sidecar 能识别、应用策略、上报遥测。

```yaml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: external-payment
spec:
  hosts: ["api.payment.com"]
  ports:
    - number: 443
      name: https
      protocol: TLS
  resolution: DNS
  location: MESH_EXTERNAL
  endpoints:
    - address: 203.0.113.10
```

**关键字段**：
- `resolution`：`NONE`（按 DNS 直连，不做 LB）、`STATIC`（固定端点）、`DNS`（定期解析）。
- `location`：`MESH_EXTERNAL` / `MESH_INTERNAL`（后者用于把 VM 加入网格）。
- `exportTo`：限制可见范围。

**默认的出站行为**：`outboundTrafficPolicy.mode`：
- `ALLOW_ANY`（默认，新版更谨慎）：未知外部服务直接放行。
- `REGISTRY_ONLY`：只允许已注册（ServiceEntry/网格内）的服务，其余拒绝。

### 5.6 Sidecar 资源

**作用**：限制某个工作负载的 Envoy 能看到哪些服务，减少配置体积与内存。

```yaml
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
  name: default
  namespace: shop
spec:
  egress:
    - hosts:
        - "payments/*"
        - "istio-system/*"
        - "~/*.external.com"
  ingress:
    - port: { number: 9080, protocol: HTTP }
  outboundTrafficPolicy:
    mode: REGISTRY_ONLY
```

**为什么重要**：默认每个 sidecar 会收到**整个网格**的 Cluster/Endpoint 配置，几千个服务时内存与配置推送成本很高。大规模集群必须用 `Sidecar` 限定范围（Istio 官方推荐）。

### 5.7 EnvoyFilter 与 WasmPlugin

- **EnvoyFilter**：直接 patch 底层 Envoy 配置（`patch.context: SIDECAR_INBOUND` 等），用于 CRD 无法表达的字段。
  - 风险：耦合 Envoy 内部结构、升级易碎、可能产生冲突。
- **WasmPlugin**：加载 Wasm 过滤器，适合自定义逻辑（鉴权、日志脱敏、A/B）。
- **ext_authz**：通过 `AuthorizationPolicy` 的 `CUSTOM` action 对接外部授权服务。

**建议顺序**：官方字段 → Gateway API → WasmPlugin → EnvoyFilter。

---

## 六、安全：身份与 mTLS

### 6.1 身份模型

每个工作负载的身份是：

```
spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>
# 例如：spiffe://cluster.local/ns/shop/sa/frontend
```

证书由 istiod（Citadel）基于 ServiceAccount 签发，**默认有效期 24 小时**，通过 SDS 自动轮换。

### 6.2 mTLS 模式

| 资源 | 作用范围 | 模式 |
| --- | --- | --- |
| `PeerAuthentication` | namespace 或工作负载 | `STRICT` / `PERMISSIVE` / `DISABLE` |
| `DestinationRule.trafficPolicy.tls` | 指定目标（客户端侧） | `ISTIO_MUTUAL` / `DISABLE` 等 |

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

**迁移建议**：先 `PERMISSIVE`（明文与 mTLS 都接受），观察监控中非 mTLS 流量，再切 `STRICT`。

**常见坑**：
- 未注入 sidecar 的服务访问 mTLS 服务会失败（`STRICT` 下）。
- Ingress 网关到后端的 mTLS 需要单独配置（`DestinationRule` 的 `ISTIO_MUTUAL` 或网关参数）。
- `PeerAuthentication` 与 `DestinationRule` 配置冲突时，以更严格/更具体者为准（工作负载级 > namespace 级 > mesh 级）。

### 6.2.1 istiod 是怎么把 mTLS 跑起来的

**先划清职责**：

```
证书谁发？        → istiod 内部的 CA（历史名称 Citadel）
加密谁做？        → Envoy 之间的 TLS 握手
istiod 参与转发吗？ → 完全不参与
```

**一句话**：istiod 是「签证机关」，Envoy 是「持证人」，加密连接由 Envoy 自己完成。

#### 证书签发链路

```
① Pod 启动，挂载 ServiceAccount token（projected volume）
        ↓
② sidecar 的 Envoy 通过 SDS 向 istiod 的 15012 端口申请证书
   （gRPC 流，请求里带 SA token 证明"我是谁"）
        ↓
③ istiod 的 CA 校验：
   · token 是否有效（TokenReview）
   · 该 SA 是否真实存在
        ↓
④ 签发 X.509 证书：
   · Subject / URI SAN = spiffe://cluster.local/ns/shop/sa/frontend
   · 有效期默认 24 小时
   · 由 Istio 内置 CA 签名（默认自签根，可替换为自有 CA / Vault / KMS）
        ↓
⑤ 通过 SDS 下发证书，私钥只在 Envoy 进程内存中
        ↓
⑥ Envoy 用它做 mTLS，并在到期前**自动续期**（无需重启 Pod）
```

**两个关键点**：
- **私钥不上传**到 istiod，istiod 只负责签名。
- 证书轮换由 SDS 完成，**不中断现有连接**（新连接用新证书）。

#### 握手时双方校验什么

```
A 的 Envoy                                    B 的 Envoy
    │──── ClientHello ──────────────────────▶ │
    │◀─── ServerHello + B 的证书 ─────────────│
    │  B 证书的 URI SAN = spiffe://.../sa/backend，      │
    │  用 istiod 的根证书验证签名                        │
    │──── A 的证书 ─────────────────────────▶ │
    │        双方校验通过 → 加密通道建立                 │
```

- **双向都校验**（这才是 mutual）。
- 校验依据是**信任根**（istiod 根证书）+ **URI SAN**，**不看 IP**。所以 Pod 换 IP、换节点，身份不变。
- Istio 会让 Envoy 只接受 SAN 符合 `spiffe://<trust-domain>/ns/*/sa/*` 的证书。

#### PERMISSIVE 是怎么做到"两种都收"的

这是迁移期不中断服务的关键。实现方式是：入站 **15006** 端口上启用 `tls_inspector` 网络过滤器，**先探测**第一个包是不是 TLS：

```
入站包 → tls_inspector 探测
            ├─ 是 TLS  → 走 mTLS filter chain（校验证书）
            └─ 不是     → 走明文 filter chain
```

切换 `STRICT` 后，明文那条 filter chain 被移除，所以非 mTLS 连接会被直接拒绝（表现为连接 reset）。

#### 故障与运维视角

| 场景 | 影响 |
| --- | --- |
| istiod 全挂 | 已签发证书继续有效、已建连接不受影响；但**证书到期时新连接会失败** |
| 证书过期 | 握手失败，报 `certificate has expired` 一类错误 |
| 迁移到 STRICT 后部分调用失败 | 通常是没有 sidecar 的客户端（Job、外来服务、hostNetwork Pod） |
| 想替换 CA | 通过 `cacerts` Secret 提供根证书与中间证书，或配置 `PILOT_CERT_PROVIDER` 对接外部 CA |

#### 检查 mTLS 是否真的生效

```bash
# 看 sidecar 拿到的证书（SAN、有效期、签发者）
istioctl proxy-config secret <pod> -n <ns>

# 看工作负载的 mTLS 状态与生效策略
istioctl x describe pod <pod> -n <ns>

# 用指标确认：只有 mutual_tls 才是真正加密的
istio_requests_total{connection_security_policy="mutual_tls"}
```

#### 面试加分点

- **"istiod 挂了 mTLS 还能用吗？"** 能，直到证书到期；到期后新连接失败，已建连接仍存活。
- **"证书泄漏怎么办？"** 证书只绑定一个 SA；撤销靠轮换 CA 根证书（全部立即失效）或等待短期证书自然过期——这也是为什么生产建议用较短有效期。
- **"Ambient 模式下机制变了吗？"** 身份与签发机制不变（同样 SDS + SPIFFE），只是**执行加密的是节点级 ztunnel**，不是每 Pod 一个 Envoy。

### 6.3 请求认证（JWT）

```yaml
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: jwt
  namespace: shop
spec:
  selector:
    matchLabels: { app: api }
  jwtRules:
    - issuer: "https://auth.example.com"
      jwksUri: "https://auth.example.com/.well-known/jwks.json"
```

**注意**：`RequestAuthentication` 只负责**校验**（失败返回 401），不负责拒绝「没有 token」的请求；拒绝要靠 `AuthorizationPolicy`。

### 6.4 授权（AuthorizationPolicy）

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: api-policy
  namespace: shop
spec:
  selector:
    matchLabels: { app: api }
  action: ALLOW          # ALLOW / DENY / AUDIT / CUSTOM
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/shop/sa/frontend"]
            requestPrincipals: ["https://auth.example.com/sub?sub=123"]
      to:
        - operation:
            methods: ["GET"]
            paths: ["/products/*"]
      when:
        - key: source.ip
          notValues: ["10.0.0.5"]
```

**语义**：
- 有 `ALLOW` 策略时，未被允许的请求会被拒绝；没有 `ALLOW` 时默认允许。
- `DENY` 优先于 `ALLOW`。
- `CUSTOM` 走外部授权服务（ext_authz）。
- 匹配维度：`source.principals`（SPIFFE）、`source.namespaces`、`source.ip`、`requestPrincipals`、`operation.methods/paths/hosts/ports`。
- `when` 条件可用 `values`/`notValues`，key 如 `source.ip`、`request.headers[x-foo]`。

**最佳实践**：先 `AUDIT` 观察，再 `ALLOW`；**显式放行 DNS/健康检查/指标端口**，否则 sidecar 自身会被拦。

---

## 七、可观测性

### 7.1 指标

| 来源 | 指标 | 用途 |
| --- | --- | --- |
| Envoy（sidecar） | `istio_requests_total`、`istio_request_duration_milliseconds`、`istio_tcp_*` | 服务间请求量、延迟、错误码 |
| 控制面 | `pilot_xds_preconnect_duration`、`pilot_xds_push_errors`、`istiod_*` | 配置下发健康度 |
| 使用方 | `istio_requests_total` 的 label | `source_workload`、`destination_workload`、`destination_service`、`response_code`、`reporter`（source/destination 双视角） |

**要点**：
- 默认指标保留在 Envoy 内存中，由 Prometheus 抓取 `:15090`。
- `reporter` 区分**客户端视角**与**服务端视角**，排查「谁报的错」很关键。
- 指标基数要控制（默认 label 已够用，自定义 label 需谨慎）。

### 7.2 访问日志

开关（`Telemetry` API 或 `meshConfig.accessLogFile`）：

```yaml
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata: { name: mesh-default, namespace: istio-system }
spec:
  accessLogging:
    - providers:
        - name: envoy
      filter:
        expression: "response.code >= 500"
```

日志字段与 Envoy 一致（见 `envoy.md`）。

### 7.3 分布式追踪

- Envoy 默认透传 `x-request-id`、W3C `traceparent`、B3 header。
- 采样率在 `meshConfig.defaultConfig.tracing.sampling` 或 `Telemetry` 中配置。
- 需要链路完整时，应用需透传 trace header（大多数 SDK 默认透传）。
- 常见后端：Jaeger、Tempo、Zipkin。

### 7.4 Kiali

Istio 官方推荐的拓扑与配置可视化工具：
- 服务依赖拓扑（实时请求量、错误率）。
- 配置校验（哪些 VS/DR 有问题）。
- 与 Prometheus/Jaeger 联动。
- 注意 Kiali 只是展示，不改变配置。

---

## 八、Ambient 模式

### 8.1 目的

解决 sidecar 的三大痛点：
1. **资源开销**：每 Pod 一个 Envoy，闲置也占内存。
2. **生命周期耦合**：升级 sidecar 要重启业务 Pod。
3. **配置与安全绕过**：应用可能绕过 sidecar（hostNetwork）。

### 8.2 架构

```
            ┌──────────────┐
            │ istiod       │  ← 控制面（支持两种模式）
            └──────┬───────┘
                   │
   ┌───────────────┴──────────────────┐
   │                                  │
   ▼                                  ▼
┌──────────────────┐         ┌────────────────────┐
│ ztunnel (DaemonSet)│       │ waypoint proxy      │
│ 每节点一个 (Rust)  │        │ 按需部署的 Envoy     │
│ - L4 mTLS         │◀──────▶│ - L7 策略/路由       │
│ - 基础遥测        │        │ - 仅需要的服务才有    │
└──────────────────┘         └────────────────────┘
        ▲
        │ 应用 Pod（无 sidecar）
        ▼
    业务容器
```

**L4 vs L7**：
- **ztunnel**：节点级 L4 代理，负责 mTLS、TCP 遥测、基本授权。
- **waypoint**：按 namespace 或 Service 部署的 L7 Envoy，负责 HTTP 路由、JWT、L7 授权、Wasm 等。

### 8.3 关键概念

- **命名空间加标签 `istio.io/dataplane-mode=ambient`** 即可让所有 Pod 被 ztunnel 纳管（无需重启 Pod）。
- 只有**需要 L7 能力的流量**才会经过 waypoint（省资源）。
- 流量劫持方式：不再用 iptables + init 容器，而是用 **eBPF / iptables 在节点层面重定向**（ztunnel 使用 node 级网络）。

### 8.4 何时选 ambient

| 场景 | 建议 |
| --- | --- |
| 只要 mTLS、基础遥测 | Ambient（ztunnel） |
| 需要 L7 灰度、JWT、L7 授权 | Ambient + waypoint |
| 需要极致兼容性 / 已有成熟 sidecar 运营 | 继续用 sidecar |
| 资源敏感（上千 Pod） | Ambient 明显更省 |

**注意**：Ambient 在某些 L7 特性上仍在追赶 sidecar，升级前要确认版本与特性覆盖。

---

## 九、一次请求的完整旅程

以 `frontend` → `reviews`（90/10 灰度、mTLS、JWT 校验）为例：

1. **应用发起**：`frontend` 访问 `http://reviews.shop:9080`。
2. **DNS**：CoreDNS 解析到 `reviews` 的 ClusterIP（或由 sidecar DNS 代理返回）。
3. **出站劫持**：iptables 把流量重定向到 `frontend` sidecar 的 `15001`。
4. **Envoy 路由**：VirtualService 生成的路由表按 host 匹配，检查 header 灰度规则，按权重选 `reviews-v1` 或 `v2`。
5. **选实例**：从 Endpoint 列表中按 `LEAST_REQUEST`（或一致性哈希）选一个 Pod IP。
6. **建立连接**：从 SDS 取证书，与目标 sidecar 做 mTLS 握手（SPIFFE 校验）。
7. **入站劫持**：目标 Pod 的 iptables 把流量重定向到 `15006`。
8. **策略执行**：目标 Envoy 校验 `AuthorizationPolicy`（服务身份）、`RequestAuthentication`（JWT），不通过则 401/403。
9. **转发应用**：转发到 `127.0.0.1:8080` 业务容器。
10. **响应回传**：业务返回 → 目标 Envoy 记录指标/日志/span → 回到源 sidecar → 源端记录延迟与错误 → 返回应用。

**排查时要知道每一层的检查点**：
- 第 3/7 步出问题 → iptables/注入；
- 第 4 步 → VirtualService 配置；
- 第 5 步 → Endpoint/DestinationRule；
- 第 6 步 → mTLS/证书；
- 第 8 步 → 授权策略。

---

## 十、安装、升级与运维

### 10.1 安装方式

```bash
# 1. istioctl（推荐入门）
istioctl install --set profile=demo -y

# 2. 生产常用 profile
istioctl install --set profile=default --set values.global.proxy.resources.requests.cpu=100m

# 3. Helm（可 GitOps）
helm install istio-base istio/base -n istio-system --create-namespace
helm install istiod istio/istiod -n istio-system --wait

# 4. 查看 profile 差异
istioctl profile diff default demo
```

**常见 profile**：`default`、`demo`（带 Kiali/Jaeger/Grafana）、`minimal`、`remote`、`empty`、`ambient`。

### 10.2 多版本与灰度升级

- **revision-based**：`istioctl install --set revision=canary`，生成 `istiod-canary`；namespace 用 `istio.io/rev: canary` 标签切换。这样可以**先让部分 namespace 用新版本**。
- sidecar 版本切换需要**重建 Pod**（注入器改的是 Pod 模板），可用 `kubectl rollout restart`。
- 控制面升级通常先于数据面；升级前 `istioctl x precheck`。

### 10.3 常用运维命令

```bash
# 状态
istioctl proxy-status
istioctl verify-install
istioctl analyze -A

# 配置生效检查
istioctl experimental describe pod <pod>
istioctl proxy-config listener <pod> --port 8080 -o json
istioctl proxy-config route <pod>
istioctl proxy-config cluster <pod>
istioctl proxy-config endpoint <pod>
istioctl proxy-config secret <pod>

# 查看注入
kubectl get mutatingwebhookconfiguration istio-sidecar-injector -o yaml

# 版本
istioctl version
kubectl get pods -n istio-system
```

### 10.4 资源规划建议

| 项 | 建议 |
| --- | --- |
| istiod | 2–3 副本，每个 500m–2C / 1–2Gi |
| ingressgateway | 2+ 副本，按流量扩（HPA） |
| sidecar | requests 100m/128Mi，limits 1000m/512Mi（按实际调） |
| 大规模 | 用 `Sidecar` 限定可见范围、开 `PILOT_ENABLE_EDS_DEBOUNCE`、考虑 ambient |

---

## 十一、排障手册

### 11.1 通用流程

```
1. 问题在哪一层？  应用 / sidecar / 控制面 / 网络
2. 配置对不对？    istioctl analyze → describe pod → proxy-config
3. 配置生效没？    proxy-status 是否 SYNCED
4. 实际转发到哪？  config_dump / envoy access log 的 %UPSTREAM_HOST%
5. 策略是否拦截？  授权日志、403 来源
```

### 11.2 常见现象与处理

| 现象 | 可能原因 | 排查 |
| --- | --- | --- |
| Pod 没有 sidecar | namespace 未打标签、Pod 注解关闭、webhook 失败 | `kubectl get ns --show-labels`、`mutatingwebhookconfiguration` |
| 503 `UF` | 上游连不上（端口/协议/mTLS） | `proxy-config endpoint`、目标端口、`PeerAuthentication` |
| 503 `NR` / `NC` | 路由或 cluster 不存在 | `proxy-config route`、VirtualService host 是否匹配 |
| 503 `UO` | 熔断溢出 | `DestinationRule.connectionPool`、`upstream_rq_pending_overflow` |
| 401 | JWT 校验失败 | `RequestAuthentication` 的 issuer/jwksUri、token 是否有效 |
| 403 | AuthorizationPolicy 拒绝 | `istioctl experimental describe`、principal 是否匹配 |
| 调用外部失败 | ServiceEntry 缺失 / `REGISTRY_ONLY` | `outboundTrafficPolicy`、ServiceEntry |
| 配置不生效 | NACK、版本未同步 | `proxy-status`（STALE/NOT SENT）、istiod 日志 |
| mTLS 握手失败 | 模式不一致、证书过期、SAN 不匹配 | `proxy-config secret`、`PeerAuthentication`、`DestinationRule.tls` |
| 启动变慢 | sidecar 等待、DNS 解析、`holdApplicationUntilProxyStarts` | 应用日志、sidecar 日志 |
| 延迟增加明显 | sidecar CPU 不足、DNS capture、重试放大 | `kubectl top pod`、`istio_requests_total`、retry 配置 |

### 11.3 抓包与调试

```bash
# 看 sidecar 出站日志（调 debug）
kubectl exec <pod> -c istio-proxy -- curl -X POST "localhost:15000/logging?connection=debug"

# 从 sidecar 侧发起请求（绕过应用）
kubectl exec <pod> -c istio-proxy -- curl -sS http://reviews.shop:9080/health

# 抓包
kubectl exec <pod> -c istio-proxy -- tcpdump -i eth0 -n port 9080

# 用 netshoot 做网络调试
kubectl debug -it <pod> --image=nicolaka/netshoot --target=app
```

### 11.4 性能问题排查

- **sidecar CPU 打满**：调大 limit、减少日志、检查是否有大量短连接（TLS 握手开销）。
- **DNS 放大**：`dnsCapture` 开启后所有 DNS 走 sidecar，高频外部域名解析会打满 CPU；用 NodeLocal DNS 或 `ndots` 优化。
- **重试风暴**：VS 的 `retries` 与上游重试叠加，用 `istio_requests_total` 的 source/destination 对比看放大倍数。
- **配置推送慢**：大规模集群里 `pilot_xds_push_time` 上升，需用 `Sidecar` 限定范围、开增量推送（Delta xDS）。

---

## 十二、面试高频问题

**Q1：Istio 是什么？和 Envoy 的关系？**
Istio 是服务网格控制面（istiod），Envoy 是数据面。istiod 监听 K8s 资源与 CRD，生成 xDS 下发给 Envoy，Envoy 负责转发、策略与遥测。**Istio 不转发流量。**

**Q2：VirtualService 和 DestinationRule 的区别？**
VS 定义「怎么路由」（匹配 + 动作，如权重/重写/重试），DR 定义「目标怎么连」（子集、LB、连接池、mTLS、outlier detection）。

**Q3：服务网格的流量为什么绕过 kube-proxy？**
sidecar 拿到的是 Endpoint 列表，Envoy 自己做 LB 选 Pod IP；Service 的 ClusterIP 只在没有 sidecar 或未按 FQDN 匹配时起作用。

**Q4：sidecar 的缺点？**
资源开销、启动顺序、升级需重建 Pod、iptables 劫持复杂度、hostNetwork 不支持。Ambient 模式用 ztunnel + waypoint 替代。

**Q5：istiod 挂掉会怎样？**
已有 sidecar 继续用本地缓存配置转发；新 Pod 无法注入、配置变更不生效、证书轮换受限。

**Q6：mTLS 怎么实现的？**
Citadel 基于 ServiceAccount 签发 SPIFFE 证书（默认 24h），SDS 下发到 Envoy，Envoy 间双向校验；`PeerAuthentication` 控制 STRICT/PERMISSIVE。

**Q7：如何做灰度发布？**
VS 权重 / header 匹配 + DR subset + 监控指标；进阶用 Flagger 或 Argo Rollouts 自动分析放量。

**Q8：AuthorizationPolicy 的默认行为？**
有 ALLOW 策略时「默认拒绝未匹配的」；没有 ALLOW 时默认允许；DENY 优先于 ALLOW。

**Q9：大规模集群怎么优化？**
`Sidecar` 限定可见服务、Delta xDS、减少大对象、指标降基数、ambient 降低 sidecar 数量、控制面多副本 + 分片。

**Q10：Istio 和 Gateway API / Ingress 的关系？**
Ingress 解决南北向 L7 路由；Istio 覆盖东西向 + 也在做入口（Gateway API 是推荐方式）。两者不是替代关系，是层次不同。

**Q11：Wasm 与 EnvoyFilter 怎么选？**
能用官方字段就不用扩展；需要自定义逻辑优先 WasmPlugin（沙箱、可热插拔）；EnvoyFilter 是最后手段，耦合内部结构。

**Q12：如何排查 Istio 中的 503？**
看 `RESPONSE_FLAGS` + `proxy-config route/cluster/endpoint` + `proxy-status` 是否同步 + mTLS 配置。

---

## 附：资源对象速查

| 资源 | API 组 | 一句话 |
| --- | --- | --- |
| VirtualService | `networking.istio.io/v1` | 路由规则 |
| DestinationRule | `networking.istio.io/v1` | 目标策略（子集/LB/连接池/mTLS） |
| Gateway | `networking.istio.io/v1` | 网关监听（旧） |
| Gateway/HTTPRoute | `gateway.networking.k8s.io` | 网关（推荐） |
| ServiceEntry | `networking.istio.io/v1` | 外部服务纳入网格 |
| Sidecar | `networking.istio.io/v1` | 限定可见范围 |
| WorkloadEntry | `networking.istio.io/v1` | 非 K8s 工作负载 |
| PeerAuthentication | `security.istio.io/v1` | mTLS 模式 |
| RequestAuthentication | `security.istio.io/v1` | JWT 校验 |
| AuthorizationPolicy | `security.istio.io/v1` | 授权 |
| Telemetry | `telemetry.istio.io/v1` | 指标/日志/追踪 |
| EnvoyFilter | `networking.istio.io/v1alpha3` | 底层补丁（慎用） |
| WasmPlugin | `extensions.istio.io/v1alpha1` | Wasm 扩展 |
| IstioOperator | `install.istio.io/v1alpha1` | 安装配置 |
| ProxyConfig | `networking.istio.io/v1beta1` | sidecar 全局参数 |
