# Bookinfo 实战实验手册

> 从零开始：用 kind 拉起集群 → 安装 Istio → 部署 bookinfo → 逐个验证灰度、超时重试、熔断、故障注入、mTLS、授权、流量镜像、可观测性。
> 每个实验都包含：**目标 → 操作 → 观察 → 结论/追问**。
> 配套阅读：[`envoy.md`](./envoy.md)、[`istio.md`](./istio.md)

---

## 目录

- [实验 0：环境与准备](#实验-0环境与准备)
- [实验 1：安装 Istio 并部署 bookinfo](#实验-1安装-istio-并部署-bookinfo)
- [实验 2：理解 sidecar 注入与流量劫持](#实验-2理解-sidecar-注入与流量劫持)
- [实验 3：入口流量与 Gateway](#实验-3入口流量与-gateway)
- [实验 4：按权重灰度（金丝雀）](#实验-4按权重灰度金丝雀)
- [实验 5：按 Header 灰度（内部测试）](#实验-5按-header-灰度内部测试)
- [实验 6：超时与重试](#实验-6超时与重试)
- [实验 7：熔断（连接池限制）](#实验-7熔断连接池限制)
- [实验 8：故障注入](#实验-8故障注入)
- [实验 9：流量镜像](#实验-9流量镜像)
- [实验 10：mTLS 与 PeerAuthentication](#实验-10mtls-与-peerauthentication)
- [实验 11：JWT 认证与授权](#实验-11jwt-认证与授权)
- [实验 12：可观测性（Kiali / Jaeger / Grafana）](#实验-12可观测性kiali--jaeger--grafana)
- [实验 13：外部服务与 ServiceEntry](#实验-13外部服务与-serviceentry)
- [实验 14：Ambient 模式对比](#实验-14ambient-模式对比)
- [实验 15：排障练习（故意制造故障）](#实验-15排障练习故意制造故障)
- [清理环境](#清理环境)
- [实验速查表](#实验速查表)

---

## 实验 0：环境与准备

### 目标
准备一台可运行 Docker 的机器（macOS/Linux），装好 `kind`、`kubectl`、`istioctl`。

### 安装工具

```bash
# kind
brew install kind                      # macOS
# 或 Linux:
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# kubectl
brew install kubectl

# istioctl（版本以官方最新稳定版为准，示例用 1.24.x）
curl -L https://istio.io/downloadIstio | sh -
export PATH="$PWD/istio-*/bin:$PATH"
istioctl version --remote=false
```

> 检查版本一致性：`istioctl version`、`kubectl version`、`kind version` 都打印出来记一笔。

### 创建集群

```bash
cat > kind-istio.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: mesh-lab
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080     # ingress gateway 的 NodePort
        hostPort: 8080
      - containerPort: 30090
        hostPort: 9090
  - role: worker
  - role: worker
EOF

kind create cluster --config kind-istio.yaml
kubectl get nodes -o wide
```

**为什么 3 个节点**：需要观察跨节点 Pod 通信、拓扑分布、节点级 DaemonSet（ambient 的 ztunnel）。

### 验证

```bash
kubectl cluster-info
kubectl get pods -A
```

**实验记录**：写下令牌版本、节点 IP 段（`kubectl get nodes -o wide` 的 INTERNAL-IP），后面抓包会用到。

---

## 实验 1：安装 Istio 并部署 bookinfo

### 目标
装上 demo profile（带 Kiali/Jaeger/Grafana），部署 bookinfo 四个服务。

### 安装

```bash
istioctl install --set profile=demo -y
kubectl get pods -n istio-system
```

### 部署 bookinfo

```bash
kubectl create namespace bookinfo
kubectl label namespace bookinfo istio-injection=enabled --overwrite

# 若在 istio 发行包目录内：
kubectl apply -n bookinfo -f samples/bookinfo/platform/kube/bookinfo.yaml

kubectl get pods -n bookinfo
```

**预期**：每个 Pod 显示 `2/2`（`app` + `istio-proxy`）。

### 验证网格内调用

```bash
kubectl exec -n bookinfo deploy/productpage-v1 -c productpage -- \
  curl -sS http://details:9080/details/1 | head -c 300
```

**观察**：能返回 JSON，说明 `details` 在网格内可访问。

### 结构化对照（bookinfo 拓扑）

```
                productpage (Python)
                 │       │       │
        ┌────────┘       │       └────────┐
        ▼                ▼                ▼
   details(Java)    reviews(Java)      ratings(Node)
                    ├─ v1（无评星）
                    ├─ v2（黑色星星，调 ratings）
                    └─ v3（红色星星，调 ratings）
```

**实验记录**：`kubectl get pods -n bookinfo --show-labels`，特别注意 `version: v1/v2/v3` 这三个 label —— 后面所有 subset 都依赖它。

---

## 实验 2：理解 sidecar 注入与流量劫持

### 目标
亲眼看到 iptables 规则、Envoy 监听端口、配置内容。

### 查看容器

```bash
POD=$(kubectl get pod -n bookinfo -l app=productpage -o jsonpath='{.items[0].metadata.name}')
kubectl get pod -n bookinfo $POD -o jsonpath='{.spec.containers[*].name}'
# 期望: productpage istio-proxy
```

### 查看 iptables 规则

```bash
kubectl exec -n bookinfo $POD -c istio-proxy -- \
  iptables -t nat -L ISTIO_OUTPUT -n -v
kubectl exec -n bookinfo $POD -c istio-proxy -- \
  iptables -t nat -L ISTIO_INBOUND -n -v
```

**观察点**：
- 有 `REDIRECT --to-ports 15001`（出站）和 `15006`（入站）。
- 有一条 `owner UID match 1337 -j RETURN` —— 避免 Envoy 自己的流量被再次劫持。

### 查看 Envoy 监听

```bash
kubectl exec -n bookinfo $POD -c istio-proxy -- ss -lntp | head -20
```

**预期端口**：`15000`（admin）、`15001`、`15006`、`15021`、`15090`、`9090`（productpage 侧监听）、可能还有 `15006` 的 tproxy。

### 查看 Envoy 拿到的配置

```bash
# 有哪些 Listener
kubectl exec -n bookinfo $POD -c istio-proxy -- \
  curl -s localhost:15000/config_dump | jq -r '.configs[] | select(."@type"|test("Listeners")) | .dynamic_listeners[].name'

# 更推荐用 istioctl
istioctl proxy-config listeners $POD -n bookinfo
istioctl proxy-config clusters  $POD -n bookinfo | grep reviews
istioctl proxy-config endpoints $POD -n bookinfo | grep reviews
```

**观察**：`reviews` 有 3 个 Cluster（v1/v2/v3）或 1 个 Cluster + subset，取决于是否配了 DestinationRule。此时还没配，应该只有服务级 cluster。

### 验证「ClusterIP 不参与转发」

```bash
# 1. 从 Pod 内访问 reviews 的 ClusterIP，看实际连到了哪个 Pod
REVIEWS_IP=$(kubectl get svc reviews -n bookinfo -o jsonpath='{.spec.clusterIP}')
kubectl exec -n bookinfo $POD -c productpage -- curl -sS http://$REVIEWS_IP:9080/reviews/0 | head -c 100

# 2. 看访问日志中的 %UPSTREAM_HOST% 是 Pod IP 还是 ClusterIP
kubectl logs -n bookinfo $POD -c istio-proxy | tail -5
```

**结论**：访问日志里 `upstream_host` 是 **Pod IP:端口**，不是 ClusterIP —— sidecar 自己做负载均衡。

### 追问
- 如果把 `productpage` 的 `sidecar.istio.io/inject` 设为 `false` 再重建，会发生什么？（会走 kube-proxy 的 ClusterIP，访问日志消失）

---

## 实验 3：入口流量与 Gateway

### 目标
通过 `istio-ingressgateway` 访问 productpage。

### 安装 gateway 配置

```bash
kubectl apply -n bookinfo -f samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl get gateway,virtualservice -n bookinfo
```

### 暴露 NodePort（kind 环境）

```bash
# 确认 gateway service 的类型与端口
kubectl get svc istio-ingressgateway -n istio-system

# 改成 NodePort 并映射 30080
kubectl patch svc istio-ingressgateway -n istio-system -p \
  '{"spec":{"type":"NodePort","ports":[{"name":"http2","port":80,"protocol":"TCP","targetPort":8080,"nodePort":30080}]}}'
```

### 访问

```bash
export INGRESS_HOST=127.0.0.1
export INGRESS_PORT=8080
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT

curl -s http://$GATEWAY_URL/productpage | grep -o '<title>.*</title>'
# 期望: <title>Simple Bookstore App</title>
```

### 观察请求链路

```bash
# 网关的访问日志
kubectl logs -n istio-system -l istio=ingressgateway --tail=20
# productpage sidecar 的日志
POD=$(kubectl get pod -n bookinfo -l app=productpage -o jsonpath='{.items[0].metadata.name}')
kubectl logs -n bookinfo $POD -c istio-proxy --tail=10
```

**观察点**：网关日志里的 `x-forwarded-for`、`upstream_host`（指向 productpage Pod IP）。

### 追问
- 网关本质是什么？（一个独立部署的 Envoy Deployment + Service，配置来自同一个 istiod）
- `bookinfo-gateway.yaml` 里 `servers.hosts` 与 VirtualService 的 `hosts` 不一致会怎样？（404 / 无匹配）

---

## 实验 4：按权重灰度（金丝雀）

### 目标
让 reviews 流量在 v1/v3 之间按 90:10 分流，并观察效果。

### 先应用默认规则（全部 v1）

```bash
kubectl apply -n bookinfo -f samples/bookinfo/networking/destination-rule-all.yaml
kubectl apply -n bookinfo -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

### 修改为 90/10

```bash
kubectl apply -n bookinfo -f - <<'EOF'
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts: [reviews]
  http:
    - route:
        - destination: { host: reviews, subset: v1 }
          weight: 90
        - destination: { host: reviews, subset: v3 }
          weight: 10
EOF
```

### 观察

```bash
# 刷新 20 次 productpage，统计红色星星出现的比例
for i in $(seq 1 20); do
  curl -s http://$GATEWAY_URL/productpage | grep -q 'color="red"' && echo v3 || echo v1
done | sort | uniq -c
```

**预期**：大约 2 次 v3，18 次 v1（请求量小时有波动）。

### 看下 Envoy 收到的配置

```bash
POD=$(kubectl get pod -n bookinfo -l app=productpage -o jsonpath='{.items[0].metadata.name}')
istioctl proxy-config routes $POD -n bookinfo --name 9080 -o json | jq '.[] | .virtualHosts[] | select(.name|test("reviews")) | .routes[].route.weightedClusters'
```

**观察**：能看到 `weightedClusters` 里两个 cluster 的权重 —— 这正是 Envoy 的 Route 配置。

### 追问
- 为什么用「权重」而不是「改副本数」？（副本比例粒度粗，且发布期间副本数波动会让比例抖动）
- 权重改为 100 后，`destination-rule-all.yaml` 里的 subset 还有用吗？（有用，仍需 subset 定义）

---

## 实验 5：按 Header 灰度（内部测试）

### 目标
内部测试人员带 `end-user: jason` 时走 v2，其他人走 v1。

```bash
kubectl apply -n bookinfo -f - <<'EOF'
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts: [reviews]
  http:
    - match:
        - headers:
            end-user:
              exact: jason
      route:
        - destination: { host: reviews, subset: v2 }
    - route:
        - destination: { host: reviews, subset: v1 }
EOF
```

### 验证

```bash
# 默认用户 -> 无星星（v1）
curl -s http://$GATEWAY_URL/productpage | grep -c 'color="black"'
# 0

# jason -> 黑色星星（v2）
curl -s -b "session=hello" -H "end-user: jason" http://$GATEWAY_URL/productpage | grep -c 'color="black"'
```

> 注意：bookinfo 的 productpage 在浏览器里通过登录功能注入 `end-user` cookie，curl 直接加 header 效果相同（因为 productpage 会把 header 透传，需确认版本行为；也可以在 productpage 上用 `-b "session=..."` 模拟）。

### 观察匹配优先级

```bash
POD=$(kubectl get pod -n bookinfo -l app=productpage -o jsonpath='{.items[0].metadata.name}')
istioctl proxy-config routes $POD -n bookinfo --name 9080 -o json | \
  jq '.[] | .virtualHosts[] | select(.name|test("reviews")) | .routes[] | {match: .match, cluster: .route.cluster}'
```

**观察**：带 header 的规则排在前面（Envoy 是按顺序匹配的）。

### 追问
- 如果同时配置了权重和 header 规则，顺序如何影响结果？（先匹配 header，未命中再按权重）
- header 名大小写敏感吗？（HTTP/2 里 header 会小写化，Envoy 匹配时通常也不区分）

---

## 实验 6：超时与重试

### 目标
用 `ratings` 服务的延迟注入能力，观察超时与重试的行为差异。

### 先恢复 v1 全部流量

```bash
kubectl apply -n bookinfo -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

### 实验 A：设置 1 秒超时

```bash
kubectl apply -n bookinfo -f - <<'EOF'
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts: [reviews]
  http:
    - route:
        - destination: { host: reviews, subset: v2 }
      timeout: 1s
EOF
```

此时 reviews v2 会调用 ratings。用 reviews v2 的延迟注入来模拟慢响应：

```bash
kubectl apply -n bookinfo -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml
kubectl get virtualservice ratings -n bookinfo -o yaml | grep -A6 fault
```

访问并计时：

```bash
time curl -s -o /dev/null -w "%{http_code}\n" http://$GATEWAY_URL/productpage
```

**预期**：约 1 秒后返回 500（`upstream request timeout`），页面显示 "Sorry, product reviews are currently unavailable"。

### 实验 B：加重试

```bash
kubectl apply -n bookinfo -f - <<'EOF'
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts: [reviews]
  http:
    - route:
        - destination: { host: reviews, subset: v2 }
      timeout: 3s
      retries:
        attempts: 3
        perTryTimeout: 1s
        retryOn: 5xx,reset,connect-failure
EOF
```

```bash
time curl -s -o /dev/null -w "%{http_code}\n" http://$GATEWAY_URL/productpage
```

**观察**：
- 总耗时约 3 秒（3 × 1s perTryTimeout），最终 500。
- 在 `ratings` 的访问日志里能看到 **3 次请求** —— 重试生效。

```bash
kubectl logs -n bookinfo -l app=ratings -c istio-proxy --tail=20 | grep -c '"GET'
```

### 实验 C：对比 source/destination 视角

```bash
# 在 Prometheus 中查看
kubectl port-forward -n istio-system svc/prometheus 9090:9090 &
# 浏览器打开 localhost:9090，执行：
#   sum(rate(istio_requests_total{destination_workload="ratings-v1"}[1m])) by (reporter)
```

**观察**：`reporter="source"` 与 `reporter="destination"` 的计数不同，差值就是重试放大的部分。

### 追问
- 为什么 `timeout` 必须大于 `perTryTimeout × attempts`？（否则整体超时会提前切断）
- 幂等性问题：POST 请求默认不该重试（`retryOn` 里要排除）。

---

## 实验 7：熔断（连接池限制）

### 目标
把 `maxConnections` 调到 1，用并发请求触发 `503 UO`。

### 配置

```bash
kubectl apply -n bookinfo -f - <<'EOF'
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
        connectTimeout: 500ms
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 100
  subsets:
    - name: v1
      labels: { version: v1 }
    - name: v2
      labels: { version: v2 }
    - name: v3
      labels: { version: v3 }
EOF
```

### 压测

```bash
# 用 fortio 或 hey（二选一）
kubectl run fortio --rm -it --image=fortio/fortio -- \
  load -c 20 -qps 100 -t 20s -loglevel Warning \
  http://reviews.bookinfo:9080/reviews/0
```

观察：
- 输出里出现大量 `503`。
- 错误信息类似 `upstream connect error or disconnect/reset before headers. reset reason: overflow`。

### 看指标

```bash
POD=$(kubectl get pod -n bookinfo -l app=productpage -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n bookinfo $POD -c istio-proxy -- curl -s localhost:15000/stats | grep -E 'pending_overflow|upstream_rq_5xx|ejections'
```

**预期**：`cluster.outbound|9080||reviews.bookinfo.svc.cluster.local.upstream_rq_pending_overflow` 增长。

### 恢复

```bash
kubectl apply -n bookinfo -f samples/bookinfo/networking/destination-rule-all.yaml
```

### 追问
- 熔断与 outlier detection 的区别？（并发上限 vs 按错误剔除实例）
- 为什么加了熔断后 `503` 也有可能是 `UH`（没有健康上游）？（outlier detection 把实例全剔了）

---

## 实验 8：故障注入

### 目标
分别注入延迟与中止，观察上游行为。

### 延迟注入

```bash
kubectl apply -n bookinfo -f samples/bookinfo/networking/fault-injection-details-delay.yaml
cat $(ls -d istio-*)/samples/bookinfo/networking/fault-injection-details-delay.yaml | head -30
```

访问并计时：

```bash
time curl -s -o /dev/null -w "%{http_code}\n" http://$GATEWAY_URL/productpage
# 约 7 秒后返回 200（延迟 7s，但未超时）
```

### 中止注入（HTTP 500）

```bash
kubectl apply -n bookinfo -f - <<'EOF'
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: details
spec:
  hosts: [details]
  http:
    - fault:
        abort:
          httpStatus: 500
          percentage:
            value: 100
      route:
        - destination: { host: details, subset: v1 }
EOF
curl -s -o /dev/null -w "%{http_code}\n" http://$GATEWAY_URL/productpage
# 500
```

### 按百分比注入

把 `percentage.value` 改成 `50`，连续请求 10 次，统计成败比例。

**观察**：`details` 的 sidecar 访问日志里 `response_code` 是 500，但 `details` 应用的日志里**没有任何请求** —— 故障在 Envoy 层就返回了。

### 恢复

```bash
kubectl delete virtualservice details -n bookinfo
kubectl apply -n bookinfo -f samples/bookinfo/networking/virtual-service-all-v1.yaml
```

### 追问
- 故障注入和真实故障的区别？（一个在代理层伪造，不会真正调用上游，适合演练与验证监控告警）
- 百分比注入如何用于灰度前验证？

---

## 实验 9：流量镜像

### 目标
把 v1 的线上流量复制一份发给 v3（不影响响应）。

```bash
kubectl apply -n bookinfo -f - <<'EOF'
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts: [reviews]
  http:
    - route:
        - destination: { host: reviews, subset: v1 }
      mirror:
        host: reviews
        subset: v3
      mirrorPercentage:
        value: 100
EOF
```

### 观察

```bash
# 请求 productpage
curl -s http://$GATEWAY_URL/productpage > /dev/null

# 看 v3 的日志：没有用户请求，但应有镜像流量
kubectl logs -n bookinfo -l app=reviews,version=v3 -c istio-proxy --tail=20 | grep reviews
```

**预期**：v3 的 sidecar 日志里有请求，但页面上不显示红色星星（响应只来自 v1）。

### 追问
- 镜像流量的响应会被丢弃吗？（会，`-shadow` 后缀的 cluster）
- 镜像适合什么场景？（新版本预验证、压测、数据回放；注意对下游产生真实副作用，如写库）

---

## 实验 10：mTLS 与 PeerAuthentication

### 目标
观察明文与 mTLS，并强制 STRICT。

### 检查当前状态

```bash
# 网格级
kubectl get peerauthentication -A
# 查看 sidecar 证书
POD=$(kubectl get pod -n bookinfo -l app=productpage -o jsonpath='{.items[0].metadata.name}')
istioctl proxy-config secret $POD -n bookinfo
```

**观察**：默认是 `PERMISSIVE`，能看到名为 `default` 的证书（`cert_chain` 与 `private_key`）。

### 验证 mTLS 生效

```bash
# 看 sidecar 之间的 TLS 状态
istioctl x describe pod $POD -n bookinfo
```

输出里有 `mTLS mode: PERMISSIVE` 与 `TLS effective` 等信息。

### 切到 STRICT

```bash
kubectl apply -n bookinfo -f - <<'EOF'
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: bookinfo
spec:
  mtls:
    mode: STRICT
EOF
```

### 验证「未注入 sidecar 的 Pod 被拒绝」

```bash
# 创建一个不带 sidecar 的 Pod
kubectl run no-sidecar --image=curlimages/curl -n bookinfo --annotations='sidecar.istio.io/inject=false' -- sleep 3600
kubectl wait --for=condition=Ready pod/no-sidecar -n bookinfo --timeout=60s

kubectl exec -n bookinfo no-sidecar -- curl -sS -o /dev/null -w "%{http_code}\n" http://details:9080/details/1
```

**预期**：连接被重置（`exit code 56` / `connection reset by peer`），因为明文流量被 STRICT 拒绝。

### 对比：带 sidecar 的 Pod

```bash
kubectl exec -n bookinfo $POD -c productpage -- curl -sS -o /dev/null -w "%{http_code}\n" http://details:9080/details/1
# 200
```

### 恢复

```bash
kubectl delete peerauthentication default -n bookinfo
kubectl delete pod no-sidecar -n bookinfo
```

### 追问
- 迁移到 STRICT 的正确顺序？（先 PERMISSIVE + 监控 `istio_requests_total{connection_security_policy!="mutual_tls"}`）
- `PeerAuthentication` 与 `DestinationRule.tls` 同时配置时谁生效？

---

## 实验 11：JWT 认证与授权

### 目标
用测试 JWT（Istio 官方示例密钥）验证 401/403 的差别。

### 准备 JWT

```bash
TOKEN=$(curl -sS "https://raw.githubusercontent.com/istio/istio/release-1.24/security/tools/jwt/samples/demo.jwt")
# 若网络不可用，可在 istio 发行包中找对应示例，或用 jwt.io 生成
```

### 配置 RequestAuthentication

```bash
kubectl apply -n bookinfo -f - <<'EOF'
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: jwt-example
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: productpage
  jwtRules:
    - issuer: "testing@secure.istio.io"
      jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.24/security/tools/jwt/samples/jwks.json"
EOF
```

### 验证「无 token 仍可访问」（关键！）

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://$GATEWAY_URL/productpage
# 200 —— RequestAuthentication 只校验，不强制
```

### 加 AuthorizationPolicy 强制要求 token

```bash
kubectl apply -n bookinfo -f - <<'EOF'
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: bookinfo
spec:
  selector:
    matchLabels:
      app: productpage
  action: ALLOW
  rules:
    - from:
        - source:
            requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
EOF
```

### 验证

```bash
curl -s -o /dev/null -w "no token: %{http_code}\n" http://$GATEWAY_URL/productpage
curl -s -o /dev/null -w "with token: %{http_code}\n" -H "Authorization: Bearer $TOKEN" http://$GATEWAY_URL/productpage
curl -s -o /dev/null -w "bad token: %{http_code}\n" -H "Authorization: Bearer bad.token" http://$GATEWAY_URL/productpage
```

**预期**：`no token: 403`、`with token: 200`、`bad token: 401`。

> **401 vs 403 的区别**：401 = token 校验失败（RequestAuthentication）；403 = 没有匹配的授权规则（AuthorizationPolicy）。

### 追问
- 为什么 `requestPrincipals` 写了两遍？（`<issuer>/<sub>`）
- 如何限制只有某个服务能调用？（`source.principals: ["cluster.local/ns/bookinfo/sa/bookinfo-productpage"]`）

### 恢复

```bash
kubectl delete requestauthentication jwt-example -n bookinfo
kubectl delete authorizationpolicy require-jwt -n bookinfo
```

---

## 实验 12：可观测性（Kiali / Jaeger / Grafana）

### 目标
把指标、日志、追踪串起来。

### 打开面板

```bash
istioctl dashboard kiali
istioctl dashboard jaeger
istioctl dashboard grafana
```

（会通过端口转发打开浏览器；kind 环境需要 `kubectl port-forward`。）

### Kiali 里能看什么

- **Graph**：选 `bookinfo`，观察 productpage → reviews → ratings/details 的实时流量、错误率。
- **Workloads / Services**：点进 `reviews`，看 VS/DR 配置与 mTLS 状态。
- **Validations**：能看到配置冲突（比如 VS host 写错）。

### Jaeger 里能看什么

- 刷新 productpage，然后按 Service=`productpage` 搜索 Trace。
- 展开一条 trace：能看到 `productpage → reviews → ratings` 的耗时分解，span 由 sidecar 生成。

### Grafana 里能看什么

- `Istio Service Dashboard`：请求量、P50/P90/P99、错误码分布。
- `Istio Workload Dashboard`：按 workload 看资源与流量。

### 查 Prometheus 指标

```bash
kubectl port-forward -n istio-system svc/prometheus 9090:9090 &
```

PromQL 示例：

```promql
# reviews 的 QPS，按 source 视角
sum(rate(istio_requests_total{destination_workload="reviews-v1", reporter="source"}[1m]))
by (response_code)

# P99 延迟
histogram_quantile(0.99,
  sum(rate(istio_request_duration_milliseconds_bucket{destination_workload="reviews-v1"}[1m]))
  by (le))
```

### 追问
- `reporter` 字段的意义？（同一请求在 source 与 destination 各记一次，排查时用 client/server 双视角）
- 为什么 Kiali 的拓扑有时缺边？（采样、未染色的流量、缺失 label）

---

## 实验 13：外部服务与 ServiceEntry

### 目标
让 sidecar 认识 `httpbin.org`，并限制为只允许已注册服务。

### 默认行为

```bash
kubectl exec -n bookinfo $POD -c productpage -- curl -s -o /dev/null -w "%{http_code}\n" https://httpbin.org/status/418
# 418（ALLOW_ANY 直接放行）
```

### 注册 ServiceEntry

```bash
kubectl apply -n bookinfo -f - <<'EOF'
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: httpbin
spec:
  hosts: ["httpbin.org"]
  ports:
    - number: 443
      name: https
      protocol: TLS
  resolution: DNS
  location: MESH_EXTERNAL
EOF
```

### 切换为仅允许注册服务

```bash
istioctl install --set profile=demo \
  --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY -y
```

### 验证

```bash
# 已注册的可用
kubectl exec -n bookinfo $POD -c productpage -- curl -s -o /dev/null -w "%{http_code}\n" https://httpbin.org/status/418
# 未注册的会被拒绝
kubectl exec -n bookinfo $POD -c productpage -- curl -s -o /dev/null -w "%{http_code}\n" https://example.com
# 503 / connection reset
```

### 恢复

```bash
istioctl install --set profile=demo --set meshConfig.outboundTrafficPolicy.mode=ALLOW_ANY -y
kubectl delete serviceentry httpbin -n bookinfo
```

### 追问
- `REGISTRY_ONLY` 是安全策略还是性能优化？（两者都有：减少配置范围、防止意外外联）
- 生产建议？（默认 REGISTRY_ONLY + 显式 ServiceEntry，符合「默认拒绝」原则）

---

## 实验 14：Ambient 模式对比

### 目标
用 ambient 模式跑同一套 bookinfo，对比 sidecar 的资源占用与配置。

> 需要支持 ambient 的 Istio 版本（1.22+ 已 GA 方向）。

### 创建 ambient 集群

```bash
istioctl install --set profile=ambient -y
kubectl get pods -n istio-system     # 应看到 ztunnel DaemonSet
```

### 让 namespace 进入 ambient

```bash
kubectl label namespace bookinfo istio-injection- --overwrite
kubectl label namespace bookinfo istio.io/dataplane-mode=ambient --overwrite

# 重启 Pod 使标签生效（ambient 下其实不需要重启，但注入标签移除后要重建）
kubectl rollout restart deploy -n bookinfo
kubectl get pods -n bookinfo         # 应为 1/1（无 sidecar）
```

### 部署 waypoint（需要 L7 时）

```bash
istioctl waypoint apply -n bookinfo --enroll-namespace
kubectl get gateway,deployment -n bookinfo
```

### 对比

```bash
# sidecar 数量
kubectl get pods -n bookinfo -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].name}{"\n"}{end}'
# ztunnel 数量
kubectl get pods -n istio-system -l app=ztunnel
# 资源
kubectl top pods -n bookinfo
```

### 追问
- ambient 下 mTLS 由谁做？（ztunnel，per-node，L4）
- 什么时候必须部署 waypoint？（需要 HTTP 路由、JWT、L7 授权、Wasm）

---

## 实验 15：排障练习（故意制造故障）

> 这一节把「制造故障 → 用工具定位 → 修复」走一遍，比看文档有效。

| # | 故障 | 制造方式 | 排查命令 | 期望看到 |
| --- | --- | --- | --- | --- |
| 1 | 503 NR | VS 的 host 改成 `reviewz` | `istioctl proxy-config route` | 没有匹配的 virtual host |
| 2 | 503 UF | 把 details 的 Service 端口改错 | `proxy-config endpoint` + `curl localhost:15000/clusters` | `upstream_cx_connect_fail` |
| 3 | 503 UO | 实验 7 的熔断配置 | `stats` 中 `pending_overflow` | 溢出计数增长 |
| 4 | 403 | 加一条只允许某个 SA 的 AuthorizationPolicy | `istioctl x describe pod` | 显示被哪条策略拒绝 |
| 5 | mTLS 失败 | 把 `PeerAuthentication` 设 STRICT，再用无 sidecar Pod 访问 | `no-sidecar` curl 报 reset | 证书握手失败 |
| 6 | 配置不生效 | 给 VS 写一个不存在的 subset | `proxy-status` + `analyze` | STALE / 配置错误提示 |
| 7 | 启动慢 | 给 sidecar 设极低 CPU limit | `kubectl top pod` | sidecar CPU 打满 |
| 8 | 延迟增加 | 打开 DNS capture + 高频外部解析 | sidecar CPU、CoreDNS 延迟 | 解析放大 |

### 诊断工具箱

```bash
# 配置体检（最有用）
istioctl analyze -n bookinfo

# 配置同步状态
istioctl proxy-status

# 某 Pod 生效的规则
istioctl x describe pod $POD -n bookinfo

# 分项配置
istioctl proxy-config listener|route|cluster|endpoint|secret $POD -n bookinfo

# 日志级别
kubectl exec -n bookinfo $POD -c istio-proxy -- curl -X POST "localhost:15000/logging?connection=debug"

# 抓包
kubectl exec -n bookinfo $POD -c istio-proxy -- tcpdump -i any -n port 9080
```

---

## 清理环境

```bash
# 删除 bookinfo
kubectl delete namespace bookinfo

# 卸载 Istio
istioctl uninstall --purge -y
kubectl delete namespace istio-system

# 删除集群
kind delete cluster --name mesh-lab
```

---

## 实验速查表

| 实验 | 核心资源 | 关键命令 | 一句话结论 |
| --- | --- | --- | --- |
| 2 劫持 | iptables | `iptables -t nat -L` | 15001/15006 + UID 1337 |
| 3 入口 | Gateway + VS | `curl $GATEWAY_URL` | 网关也是 Envoy |
| 4 权重灰度 | VS + DR subset | `proxy-config routes` | 路由在 Envoy 里是 weightedClusters |
| 5 Header 灰度 | VS match | `-H "end-user: jason"` | 按顺序匹配 |
| 6 超时重试 | VS timeout/retries | `time curl` | 总超时要大于 perTryTimeout×attempts |
| 7 熔断 | DR connectionPool | fortio 压测 | 503 UO = 溢出 |
| 8 故障注入 | VS fault | `abort/delay` | 在 Envoy 层伪造，上游不会收到 |
| 9 镜像 | VS mirror | 看 v3 日志 | 响应被丢弃 |
| 10 mTLS | PeerAuthentication | `proxy-config secret` | STRICT 下无 sidecar 被拒 |
| 11 JWT | RequestAuthentication + AuthorizationPolicy | 401 vs 403 | 前者只校验，后者才拒绝 |
| 12 观测 | Kiali/Jaeger/Grafana | `istioctl dashboard` | source/destination 双视角 |
| 13 外联 | ServiceEntry + REGISTRY_ONLY | curl 外部域名 | 默认拒绝已注册之外 |
| 14 Ambient | ztunnel + waypoint | `istioctl waypoint apply` | 无 sidecar，L4 由节点代理 |
| 15 排障 | - | `analyze` / `proxy-status` | 先看配置是否同步 |
