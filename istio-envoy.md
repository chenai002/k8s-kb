# Envoy 与 Istio 快速入门

> 一篇能让你在 30 分钟内「听懂并对讲」的文档。
> 目标：搞清楚 **Envoy 是什么**、**Istio 是什么**、**两者什么关系**、**控制面和数据面怎么协作**，以及面试/实践中会被问到的关键点。

---

## 目录

- [一、一句话理解](#一一句话理解)
- [二、先搞懂 Envoy](#二先搞懂-envoy)
  - [2.1 Envoy 解决了什么问题](#21-envoy-解决了什么问题)
  - [2.2 线程与架构模型](#22-线程与架构模型)
  - [2.3 核心概念：Listener / Filter / Cluster / Router](#23-核心概念listener--filter--cluster--router)
  - [2.4 配置是静态的，但可以被动态下发](#24-配置是静态的但可以被动态下发)
  - [2.5 为什么是 Envoy 而不是 Nginx](#25-为什么是-envoy-而不是-nginx)
- [三、再搞懂 Istio](#三再搞懂-istio)
  - [3.1 Istio 解决的问题](#31-istio-解决的问题)
  - [3.2 架构：控制面 + 数据面](#32-架构控制面--数据面)
  - [3.3 流量如何被劫持](#33-流量如何被劫持)
  - [3.4 核心资源对象](#34-核心资源对象)
- [四、两者关系](#四两者关系)
- [五、一次请求的完整旅程](#五一次请求的完整旅程)
- [六、常见使用场景与配置](#六常见使用场景与配置)
- [七、面试高频问题](#七面试高频问题)
- [八、学习路径与排障命令](#八学习路径与排障命令)
- [附：术语速查表](#附术语速查表)

---

## 一、一句话理解

| 名字 | 一句话 | 类比 |
| --- | --- | --- |
| **Envoy** | 一个用 C++ 写的高性能 **L4/L7 代理**，可被远程动态配置 | 一个「可编程的 Nginx」 |
| **Istio** | 一个 **服务网格** 平台，用 Envoy 作为数据面，自己提供控制面去统一配置所有代理 | 一个「Envoy 集群的管理大脑 + 规则语言」 |

再压缩成一句：

> **Istio 把 Envoy 当作积木，用它搭建了一张覆盖所有微服务的「智能网络层」，让流量治理从「写代码」变成「写 YAML」。**

---

## 二、先搞懂 Envoy

### 2.1 Envoy 解决了什么问题

在微服务里，几乎每个服务都需要：超时、重试、熔断、限流、TLS、灰度、指标、链路追踪。如果每个语言、每个团队各写一套 SDK：

- 语言碎片化（Java/Go/Python 各一套）；
- 升级要改代码、发版；
- 行为不一致，排障困难；
- 业务代码被基础设施污染。

**Envoy 的思路**：把这些能力下沉到一个「进程外代理」（sidecar），业务代码只连 `localhost`，其余交给代理。

它提供：
- L4（TCP）与 L7（HTTP/1.1、HTTP/2、gRPC）代理；
- 健康检查、负载均衡、重试、超时、熔断、限流；
- TLS 终止与 mTLS 发起；
- 指标（Prometheus）、访问日志、分布式追踪（W3C/B3 header）；
- 动态配置（xDS），可热更新而不重启。

### 2.2 线程与架构模型

- **多线程 + 每线程一个事件循环**（libevent 风格），不是「每连接一线程」。
- 一个 **Main 线程**负责配置管理与 xDS 更新，**Worker 线程**各自处理连接，配置通过线程本地存储（TLS slot）无锁共享。
- 非阻塞、异步、基于回调链（filter chain）。
- 结果：高并发、低延迟、CPU 密集度低，但也意味着**调试是异步的**，日志与配置排障需要经验。

### 2.3 核心概念：Listener / Filter / Cluster / Router

一条请求在 Envoy 里的路径：

```
下游连接
  └── Listener（监听地址，如 0.0.0.0:15006）
       └── Filter Chain（一组过滤器，按顺序执行）
            ├── Network Filter（L4，如 TCP proxy / mTLS）
            └── HTTP Connection Manager（HCM，L7 入口）
                 └── HTTP Filter（如 jwt_authn、fault、router）
                      └── Route（匹配 host/path/header，决定去哪）
                           └── Cluster（一组上游端点 + 负载均衡策略）
                                └── Endpoint（具体 IP:Port）
```

关键对象：

| 概念 | 作用 | 记忆点 |
| --- | --- | --- |
| **Listener** | 监听端口与协议 | 「门」 |
| **Filter Chain** | 处理连接/请求的插件链 | 「门口的安检流程」 |
| **Route** | 按条件匹配并选择 Cluster | 「路由表」 |
| **Cluster** | 上游服务的逻辑分组 | 「目的地集合」 |
| **Endpoint** | Cluster 里的具体实例 | 「具体哪台机器」 |
| **Virtual Host** | 按域名分组的路由集合 | 「按域名分表」 |

负载均衡策略：Round Robin、Least Request、Random、Ring Hash（一致性哈希）、Maglev、Weighted。还支持 `outlierDetection`（离群点检测，类似被动熔断）、`circuitBreakers`（连接/请求数上限）、`retryPolicy`、`timeout`。

### 2.4 配置是静态的，但可以被动态下发

Envoy 有两种配置方式：

1. **静态配置**（bootstrap 或 `--config-path`）：启动时加载，改配置要重启。
2. **动态配置（xDS）**：Envoy 作为客户端连上一个「配置服务器」，按需订阅、增量更新。

**xDS 协议族**：

| 缩写 | 全称 | 管什么 |
| --- | --- | --- |
| **LDS** | Listener Discovery Service | 监听器 |
| **RDS** | Route Discovery Service | 路由 |
| **CDS** | Cluster Discovery Service | 上游集群 |
| **EDS** | Endpoint Discovery Service | 集群里的实例 |
| **SDS** | Secret Discovery Service | 证书与密钥 |
| **VHDS** | Virtual Host Discovery Service | 虚拟主机（按需） |

这些都是 gRPC 流式协议，支持 **ACK/NACK**：Envoy 收到配置会校验，失败就 NACK 并保留旧配置——这是「配置错了不会立刻炸」的重要机制。

> **关键理解**：Envoy 本身「不会自己发现服务」。它只能接收别人下发的 Listeners/Routes/Clusters/Endpoints。**谁下发？——在 Istio 里就是 istiod。**

### 2.5 为什么是 Envoy 而不是 Nginx

| 维度 | Envoy | Nginx |
| --- | --- | --- |
| 设计目标 | 服务间通信（东西向），多租户、动态 | 边缘网关/静态站点（南北向），配置驱动 |
| 动态配置 | xDS 原生支持，热更新 | 商业版有 API，开源需 reload |
| 协议 | HTTP/2、gRPC 原生，支持 HTTP/3 | HTTP/2 支持较晚，gRPC 需额外配置 |
| 可观测性 | 指标/日志/追踪内建，结构统一 | 需模块或第三方 |
| 扩展 | C++/Lua/Wasm 过滤器 | C 模块、Lua |
| 单进程性能 | 优秀，多线程 | 优秀，但 reload 有抖动 |

结论：**Envoy 不是「更好的 Nginx」，而是为「大量服务互相调用、配置频繁变化」这个场景设计的。**

---

## 三、再搞懂 Istio

### 3.1 Istio 解决的问题

有了 Envoy，每个服务都有能力做重试、熔断、mTLS、灰度。但新的问题是：

- 几百上千个 Envoy 谁来配置？
- 配置如何保持一致？
- 如何知道谁在跟谁说话（拓扑）？
- 如何做统一的身份认证与授权？
- 如何按「服务」而不是「IP」写策略？

**Istio 就是这套「配置 + 身份 + 观测 + 策略」的控制面。**

### 3.2 架构：控制面 + 数据面

```
                        ┌───────────────────────────────┐
                        │       Istio 控制面             │
                        │                               │
   用户 / GitOps ──────▶ │  istiod                       │
   (VirtualService,     │   ├─ Pilot   (xDS 下发)       │
    DestinationRule,    │   ├─ Citadel (证书签发/mTLS)  │
    AuthorizationPolicy)│   └─ Galley→配置校验(旧版)     │
                        └──────────────┬────────────────┘
                                       │ xDS (gRPC)
              ┌────────────────────────┼────────────────────────┐
              ▼                        ▼                        ▼
      ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
      │  Node A      │         │  Node B      │         │  Node C      │
      │ ┌──────────┐ │         │ ┌──────────┐ │         │ ┌──────────┐ │
      │ │ App Pod  │ │         │ │ App Pod  │ │         │ │ App Pod  │ │
      │ │  + Envoy │ │◀──mTLS─▶│ │  + Envoy │ │◀──mTLS─▶│ │  + Envoy │ │
      │ │ (sidecar)│ │         │ │ (sidecar)│ │         │ │ (sidecar)│ │
      │ └──────────┘ │         │ └──────────┘ │         │ └──────────┘ │
      └──────────────┘         └──────────────┘         └──────────────┘
                 数据面：Envoy（也可能用「无 sidecar」的 Ambient 模式）
```

- **控制面（istiod）**：监听 Kubernetes 里的 Service、EndpointSlice、Istio CRD，翻译成 Envoy 的 xDS 配置，推给所有 sidecar；同时负责签发工作负载证书。
- **数据面（Envoy）**：实际转发流量、执行策略、上报遥测。
- **控制面不参与数据转发**：istiod 挂了，已有流量不受影响，只是不能更新配置、不能签发新证书。

### 3.3 流量如何被劫持

Istio 的 sidecar 注入后，Pod 里有：

| 端口 | 用途 |
| --- | --- |
| 15000 | Envoy 管理端口（admin） |
| 15001 | 出站流量劫持（Outbound） |
| 15006 | 入站流量劫持（Inbound） |
| 15020 | 合并指标与健康检查 |
| 15021 | 就绪/存活探针 |
| 15090 | Prometheus 指标 |

劫持原理：
1. `istio-init`（或 iptables 规则）在 Pod 的 network namespace 里加 iptables/nftables 规则。
2. **出站**：所有非 Envoy 进程的流量重定向到 `15001`，Envoy 根据目标 IP/域名查 Cluster，再转发出去。
3. **入站**：所有入站流量重定向到 `15006`，Envoy 处理后再转给本机业务端口（通常 `127.0.0.1:8080`）。
4. 业务容器完全不知道代理的存在，通过 `localhost` 访问「外部服务」。

### 3.4 核心资源对象

| 资源 | 作用 | 典型用法 |
| --- | --- | --- |
| **VirtualService** | 路由规则：按 host/path/header/权重匹配，决定去哪 | 灰度、重定向、重写、超时、重试 |
| **DestinationRule** | 目标服务的策略：负载均衡、子集（subset）、连接池、mTLS、离群检测 | 定义 `v1`/`v2` 子集、一致性哈希 |
| **Gateway** | 入口网关的监听配置（端口、TLS、host） | 南北向入口 |
| **ServiceEntry** | 把外部服务注册进网格（如 `api.example.com`、数据库） | 让 sidecar 认识外部依赖 |
| **Sidecar** | 限制某工作负载能看到哪些服务（减少配置量） | 大集群性能优化 |
| **AuthorizationPolicy** | L4/L7 授权：允许/拒绝哪些请求 | 「只允许 frontend 调 api 的 GET」 |
| **RequestAuthentication** | 校验 JWT | 对接 OIDC |
| **PeerAuthentication** | 工作负载间 mTLS 模式（STRICT/PERMISSIVE/DISABLE） | 全网格强制 mTLS |
| **Telemetry** | 指标/日志/追踪的配置（新 API） | 调整采样率、标签 |
| **EnvoyFilter** | 直接给 Envoy 打补丁（底层逃生口） | 特殊需求，慎用 |
| **WasmPlugin** | 用 WebAssembly 写过滤器 | 自定义逻辑 |

**必须理解的两条**：`VirtualService` 管「怎么路由」，`DestinationRule` 管「到了之后怎么连」。

一个最小例子（按权重灰度）：

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts: ["reviews"]
  http:
    - route:
        - destination: { host: reviews, subset: v1 }
          weight: 90
        - destination: { host: reviews, subset: v2 }
          weight: 10
---
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
    - name: v1
      labels: { version: v1 }
    - name: v2
      labels: { version: v2 }
```

---

## 四、两者关系

用三个层次理解：

**1. 依赖关系**
Istio 的默认数据面就是 Envoy。没有 Envoy，Istio 也可以换成别的代理（历史上支持过 nginx、linkerd 的某些组件），但今天 Istio = istiod + Envoy。

**2. 分工关系**

| | Envoy | Istio (istiod) |
| --- | --- | --- |
| 是什么 | 代理进程 | 控制面服务 |
| 管什么 | 转发、策略执行、遥测上报 | 配置生成、下发、证书签发 |
| 输入 | xDS 配置 | Kubernetes API + Istio CRD |
| 出了故障 | 该 Pod 的流量受影响 | 新配置不生效，已有流量正常 |
| 扩展方式 | Filter / Wasm | CRD / WasmPlugin / EnvoyFilter |

**3. 抽象层次**
- Envoy 的配置是「机器视角」：Listener、Cluster、Endpoint、Filter。
- Istio 的配置是「服务视角」：VirtualService、DestinationRule、Policy。
- **istiod 的核心工作就是把「服务视角」翻译成「机器视角」的 xDS**，并根据 Endpoint 变化实时更新。

> 面试金句：**「Istio 不转发流量，Envoy 才转发；istiod 只是把声明式规则编译成各代理的配置。」**

---

## 五、一次请求的完整旅程

假设 `frontend` 调用 `reviews:9080`，配置了 90/10 灰度：

1. `frontend` 发起 `http://reviews:9080` 的请求（DNS 由 CoreDNS 解析，可能返回 `reviews` 的 ClusterIP）。
2. iptables 把出站流量劫持到 Envoy 的 `15001`。
3. `frontend` 的 Envoy 根据 VirtualService 生成的 Route 做匹配（host/path/header），按权重 90:10 选择 `v1` 或 `v2` 的 Cluster。
4. Envoy 从 Endpoint 列表里按负载均衡策略选一个 Pod IP（注意：**Envoy 直连 Pod IP，绕开 kube-proxy 的 Service VIP**）。
5. 发起 mTLS 连接：从 SDS 拿到证书，和目标 Envoy 做双向认证。
6. 目标 Node 的 iptables 把入站流量劫持到 `15006`。
7. 目标 Envoy 执行 AuthorizationPolicy、限流、路由，再转给业务容器 `127.0.0.1:9080`。
8. 业务返回，目标 Envoy 记录指标/日志/追踪，响应原路返回。
9. 如果配置了重试/超时/熔断，都在 Envoy 这一层生效，业务代码无感知。

**值得记住的点**：Service VIP 在这一步并不参与转发（除非没装 sidecar 或用了 `PassthroughCluster`），这会影响你对「负载均衡在哪一层生效」的判断。

---

## 六、常见使用场景与配置

### 6.1 金丝雀发布（按权重）

见上文 VirtualService + DestinationRule 示例。观察指标后逐步调权重到 100%。

### 6.2 按 Header 灰度（内部测试）

```yaml
http:
  - match:
      - headers:
          x-canary: { exact: "true" }
    route:
      - destination: { host: reviews, subset: v2 }
  - route:
      - destination: { host: reviews, subset: v1 }
```

### 6.3 超时 / 重试 / 熔断

```yaml
# VirtualService 里
http:
  - timeout: 2s
    retries:
      attempts: 3
      perTryTimeout: 500ms
      retryOn: 5xx,reset,connect-failure
---
# DestinationRule 里
trafficPolicy:
  connectionPool:
    tcp: { maxConnections: 100 }
    http: { http1MaxPendingRequests: 50, http2MaxRequests: 200 }
  outlierDetection:
    consecutive5xxErrors: 5
    interval: 30s
    baseEjectionTime: 60s
```

### 6.4 强制 mTLS

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT   # 也可以 PERMISSIVE 做渐进迁移
```

### 6.5 访问控制

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend
  namespace: shop
spec:
  selector:
    matchLabels: { app: api }
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/shop/sa/frontend"]
      to:
        - operation:
            methods: ["GET"]
            paths: ["/products/*"]
```

### 6.6 接入外部服务

```yaml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: external-api
spec:
  hosts: ["api.partner.com"]
  ports:
    - number: 443
      name: https
      protocol: TLS
  resolution: DNS
  location: MESH_EXTERNAL
```

### 6.7 用 Ingress / Gateway 暴露服务

从 1.22 起，Istio 推荐用 **Kubernetes Gateway API**（`Gateway` + `HTTPRoute`），老的 `istio.io/v1alpha3 Gateway` 仍可用。入口网关本质就是一个独立部署的 Envoy Deployment（`istio-ingressgateway`）。

---

## 七、面试高频问题

**Q1：Istio 是什么？和 Envoy 什么关系？**
Istio 是服务网格控制面，Envoy 是数据面代理。Istio 监听 K8s 资源并生成 xDS 下发给 Envoy，Envoy 负责实际转发。

**Q2：sidecar 模式有什么问题？**
资源开销（每个 Pod 多一个容器，内存约 50–100MB）、启动顺序与延迟、升级需重启业务 Pod、iptables 劫持增加复杂度。**Ambient 模式**（1.21 起 alpha，已 GA 方向）用「per-node ztunnel（L4）+ 按需 waypoint proxy（L7）」替代 sidecar，解决这些问题。

**Q3：为什么不用 Service 的 ClusterIP 做负载均衡？**
Envoy 直接拿到 Endpoint 列表自己选后端，可以按权重、一致性哈希、区域亲和等策略精细控制；kube-proxy 只能做简单转发，且多一跳。

**Q4：控制面挂了会怎样？**
已有 sidecar 的流量继续工作（配置在本地缓存）；新 Pod 注入、配置变更、证书轮换会受影响。通常部署 2+ istiod 副本。

**Q5：mTLS 是怎么实现的？**
istiod（Citadel）为每个工作负载的 ServiceAccount 签发证书（有效期默认 24h，自动轮换），通过 SDS 下发给 Envoy；Envoy 之间用双向 TLS 通信，身份由 SPIFFE ID（`spiffe://cluster.local/ns/<ns>/sa/<sa>`）表示。

**Q6：VirtualService 和 DestinationRule 的区别？**
前者是「路由/动作」，后者是「目标策略」（子集、负载均衡、连接池、mTLS）。

**Q7：Envoy 如何热更新配置？**
xDS 流式订阅，Envoy 校验后 ACK，失败 NACK 并保留旧配置；不需要重启进程。

**Q8：如何排查 sidecar 注入失败？**
检查 namespace 的 `istio-injection=enabled`、Pod 的 `sidecar.istio.io/inject` 注解、webhook 是否可用、Pod securityContext 是否阻止 init 容器。

**Q9：Istio 与 Ingress Controller 的区别？**
Ingress 主要解决南北向入口；Istio 覆盖东西向服务间通信（且也能做入口）。Ingress 是 L7 路由，网格还包含身份、策略、观测、流量治理。

**Q10：为什么要用 EnvoyFilter？什么时候不该用？**
当 CRD 表达不了某个 Envoy 字段时用；但它耦合了 Envoy 内部结构，升级易碎，优先用 Gateway API / WasmPlugin / 官方字段。

---

## 八、学习路径与排障命令

### 8.1 学习路线（建议顺序）

1. 先看 Envoy 的官方「What is Envoy」+ 跑一个静态配置 demo（`docker run envoyproxy/envoy`）。
2. 理解 Listener → Filter → Route → Cluster 这条链，能读懂一份 `envoy.yaml`。
3. 装一个本地 Istio（kind/minikube + `istioctl install --set profile=demo`）。
4. 部署 bookinfo，改 VirtualService 观察权重、超时、重试的行为。
5. 开 Kiali 看拓扑，开 Jaeger 看链路。
6. 再深入：`istioctl proxy-config` 看 xDS、mTLS 抓包、Ambient 模式。

### 8.2 常用命令

```bash
# 安装与状态
istioctl install --set profile=demo -y
istioctl verify-install
istioctl proxy-status                    # 所有 sidecar 的配置同步状态

# 看某个 Pod 实际拿到的配置
istioctl proxy-config listeners <pod> -n <ns>
istioctl proxy-config routes    <pod> -n <ns>
istioctl proxy-config clusters  <pod> -n <ns>
istioctl proxy-config endpoints <pod> -n <ns>
istioctl proxy-config secret    <pod> -n <ns>   # 证书

# 配置体检
istioctl analyze -n <ns>                 # 找出冲突/无效配置
istioctl experimental describe pod <pod> # 看生效的规则

# Envoy 自身
kubectl exec <pod> -c istio-proxy -- curl localhost:15000/config_dump
kubectl exec <pod> -c istio-proxy -- curl localhost:15000/clusters | head

# 查看注入与版本
kubectl get mutatingwebhookconfiguration
kubectl get pods -n <ns> -o jsonpath='{..image}'
```

### 8.3 排障思路

| 现象 | 先查什么 |
| --- | --- |
| Pod 没有 sidecar | namespace label、注入注解、webhook 状态 |
| 调用 503 `UF` / `NR` | `proxy-config clusters` 看有没有 endpoint；`proxy-config endpoints` 看健康 |
| 调用 502 | 后端未监听、协议不匹配（HTTP vs gRPC）、端口错 |
| mTLS 握手失败 | `PeerAuthentication` 模式不一致、证书过期、SDS 未下发 |
| 配置改了不生效 | `istioctl proxy-status` 是否 SYNCED；`analyze` 找冲突 |
| 延迟变高 | sidecar 资源不足、DNS 查询放大（`dnsCapture`）、重试放大 |

---

## 附：术语速查表

| 术语 | 含义 |
| --- | --- |
| **Service Mesh** | 把服务间通信能力下沉到基础设施层的架构 |
| **Data Plane** | 实际处理流量的代理（Envoy） |
| **Control Plane** | 生成并下发配置的服务（istiod） |
| **Sidecar** | 与业务容器同 Pod 的代理容器 |
| **Ambient** | 无 sidecar 模式：ztunnel（L4）+ waypoint（L7） |
| **xDS** | Envoy 的动态配置协议族（LDS/RDS/CDS/EDS/SDS） |
| **HCM** | HTTP Connection Manager，Envoy 的 L7 入口 |
| **Cluster** | Envoy 中上游服务的逻辑分组 |
| **Endpoint** | Cluster 中的一个具体实例地址 |
| **Subset** | DestinationRule 中按 label 划分的服务子集 |
| **VirtualService** | 路由规则（怎么走） |
| **DestinationRule** | 目标策略（怎么连） |
| **SPIFFE ID** | 网格内工作负载身份，`spiffe://cluster.local/ns/<ns>/sa/<sa>` |
| **mTLS** | 双向 TLS，网格内的默认加密与身份校验方式 |
| **Outlier Detection** | 被动健康检查，剔除异常实例（类似熔断） |
| **Waypoint** | Ambient 模式下按需部署的 L7 代理 |
| **ztunnel** | Ambient 模式下的 per-node L4 代理（Rust 编写） |
| **Wasm** | WebAssembly，Envoy/Istio 的扩展方式 |
