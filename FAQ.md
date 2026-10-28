# Kubernetes 面试 FAQ（100 题·详解版）

> 面向中高级 Kubernetes / 云原生岗位的面试准备材料。
> 每题包含：**核心答案要点** → **展开说明**（原理 / 机制 / 细节）→ **面试加分点**（追问、坑点、实战）。
> 建议用法：先看题目自己口述 1–2 分钟，再对照展开部分补细节；面试时讲「要点 + 一个例子」即可，不必背全。

---

## 目录

- [一、基础概念（1–15）](#一基础概念115)
- [二、架构与组件（16–30）](#二架构与组件1630)
- [三、工作负载与配置（31–45）](#三工作负载与配置3145)
- [四、网络（46–60）](#四网络4660)
- [五、存储（61–70）](#五存储6170)
- [六、安全（71–82）](#六安全7182)
- [七、可观测性与运维（83–92）](#七可观测性与运维8392)
- [八、故障排查实战（93–97）](#八故障排查实战9397)
- [九、生产实践与进阶（98–100）](#九生产实践与进阶98100)
- [附：高频追问速记](#附高频追问速记)

---

## 一、基础概念（1–15）

### 1. Kubernetes 是什么？解决什么问题？

**要点**：容器编排平台，把「一堆机器」抽象成一个统一的算力池，让用户用声明式的方式描述应用，由系统负责落地和维持。

**展开**：
- **调度**：把 Pod 放到最合适的节点（资源、亲和性、污点、拓扑）。
- **自愈**：容器崩溃重启、Pod 被删重建、节点失联后在其他节点重建。
- **服务发现与负载均衡**：Service + DNS，提供稳定虚拟 IP 和域名。
- **发布与回滚**：Deployment 的滚动更新、`rollout undo`。
- **配置与密钥管理**：ConfigMap、Secret。
- **存储编排**：PV/PVC 按需挂载块存储、文件存储、对象存储。
- **横向扩展**：HPA/VPA、Cluster Autoscaler。
- **多租户与权限**：Namespace、RBAC、ResourceQuota。

**面试加分点**：
- K8s 本质是「声明式 API + 控制器循环 + 可扩展 API」，不是一个简单的自动化脚本集合。
- 它不做的事：不打包镜像（那是 CI 的事）、不管应用的业务逻辑、不是数据库即服务、不负责机器本身的供给（那是 IaC / 云平台的事）。
- 代价也要提：控制面复杂度、etcd 成为单点关注、网络与存储的抽象泄漏、YAML 膨胀。面试官喜欢听「什么时候不该上 K8s」。

### 2. 声明式 API 与命令式 API 的区别？

**要点**：命令式描述「动作」，声明式描述「目标」，由控制器不断调和。

**展开**：
- 命令式：`kubectl run nginx --image=nginx`、`kubectl scale --replicas=3`，执行一次就结束，不可重放、不可审计。
- 声明式：`kubectl apply -f deploy.yaml`，把 YAML 提交给 apiserver，控制器（Deployment controller）比对期望副本数与实际副本数，决定扩/缩。
- `kubectl apply` 还维护 `last-applied-configuration` 注解（客户端三路合并），所以能算出「用户改了什么」；`kubectl create/replace` 是整对象覆盖。
- 服务端 apply（SSA，1.16+ GA）把合并逻辑移到 apiserver，用 `managedFields` 记录各字段的负责人，支持多个控制器/工具共同管理同一对象而不互相覆盖。

**面试加分点**：
- 声明式的好处：幂等、可 Git 化、可审计、可自愈（人为删掉 Pod 也会被拉回来）。
- 坑：`kubectl apply` 和 `kubectl edit` 混用会导致字段归属混乱；SSA 下 field manager 冲突是常见排查点。
- 与 GitOps 的关系：Git 存期望状态，Argo CD/Flux 在集群内持续 apply 并检测漂移。

### 3. Pod 是什么？为什么最小调度单位是 Pod 而不是容器？

**要点**：Pod 是一组「共享网络命名空间和存储卷」的容器集合，是调度的原子单位。

**展开**：
- Pod 内的容器共享：network namespace（同一 IP、同一个 `localhost`、同一端口空间）、IPC namespace（可选）、volumes、cgroup 资源上限（在 Pod 级别）、ServiceAccount 与调度属性。
- 共享方式：kubelet 先创建 **sandbox（pause 容器）**，建立 network/IPC namespace，其他容器通过 `join` 进入。pause 容器持有网络命名空间，所以主容器重启时 Pod IP 不变。
- 为什么不调度单个容器：真实应用常需要「主容器 + 强耦合的辅助容器」（sidecar 代理、日志采集、配置同步），它们必须同机部署、共享 localhost、整体扩缩。若以容器为单位调度，就需要自己实现协同调度。
- Pod 的字段都是「不可变」或「重建才生效」：改镜像会滚成新 Pod；改资源 requests 通常需要重建；Pod 不支持原地修改大部分 spec（1.27+ 有 in-place 资源调整 alpha/beta 能力）。

**面试加分点**：
- Pod 本身是「一次性」的（ephemeral）：不重启不迁移，稳定的是控制器的抽象（Deployment/StatefulSet）。
- Pod 内有多个容器时，整体一起调度、一起销毁；资源 requests 是容器级之和。
- 多容器 Pod 的缺点是粒度粗、升级不能独立、日志混淆，所以不要把「松耦合」服务塞进一个 Pod。

### 4. ReplicaSet、Deployment、StatefulSet、DaemonSet 的区别？

**要点**：四类控制器对应四种「副本语义」。

**展开**：
- **ReplicaSet**：维持 `replicas` 数量的 Pod。它的 selector 是 `matchLabels`，Pod 由 `ownerReferences` 反向确认归属。一般不直接使用，由 Deployment 创建。
- **Deployment**：管理 ReplicaSet 的版本演进。`strategy.type: RollingUpdate` 时用 `maxSurge`（最多超出期望值的数量/百分比）和 `maxUnavailable`（最多不可用数量）控制节奏；每个新 Pod Ready 才继续。`revisionHistoryLimit` 保留旧 RS 用于回滚。
- **StatefulSet**：`serviceName` 提供稳定域名 `<pod>.<svc>.<ns>.svc`，`volumeClaimTemplates` 为每个 Pod 生成独立 PVC，Pod 序号固定（`-0/-1/-2`），默认**顺序**创建、**逆序**删除；`podManagementPolicy: Parallel` 可打乱顺序。适合有主从/分片语义的服务（MySQL、Kafka、Elasticsearch）。
- **DaemonSet**：每个匹配的节点运行一个 Pod，默认对 `nodeSelector`/affinity 过滤后的所有节点生效；滚动更新用 `updateStrategy: RollingUpdate`（注意 `maxUnavailable` 默认 1）。适合日志 Agent、监控 Agent、CNI、CSI node 插件。System 关键 Pod 的容忍度（如容忍 `node.kubernetes.io/disk-pressure`）需要显式写。

**面试加分点**：
- Deployment 不是「保证唯一副本」，扩缩容时可能短暂超出（surge）。
- StatefulSet 的 Pod 重建后 IP 会变，但 DNS 名和 PVC 不变；**它不解决数据一致性**，一致性要靠应用或 Operator。
- DaemonSet 与节点 taint 的交互经常考：节点 NotReady/DiskPressure 时 daemonset Pod 也会被驱逐，除非加 toleration；这也是「kubectl drain 后日志 agent 没了」的原因。

### 5. Job 和 CronJob 的区别？

**要点**：Job 是「跑到成功为止的一次性任务」，CronJob 是「按时间表生成 Job」。

**展开**：
- Job 关键字段：`completions`（成功次数）、`parallelism`（并发数）、`backoffLimit`（重试上限）、`activeDeadlineSeconds`（总超时）、`ttlSecondsAfterFinished`（自动清理）、`restartPolicy` 只能是 `OnFailure`/`Never`。
- 两种模型：**非索引 Job**（每个 Pod 做同样的工作）和 **索引化 Job**（1.27 beta，`completionMode: Indexed`，每个 Pod 拿到 `JOB_COMPLETION_INDEX`，适合分片处理）。
- CronJob 关键字段：`schedule`（5 段 cron，注意时区默认 UTC，1.27+ 支持 `timeZone`）、`concurrencyPolicy`（Allow / Forbid / Replace）、`startingDeadlineSeconds`（错过窗口多久内还能补跑）、`successfulJobsHistoryLimit`/`failedJobsHistoryLimit`。
- 常见坑：`concurrencyPolicy: Allow` 默认会让长任务重入；`Forbid` 下任务超时会导致所有后续窗口被跳过；用 `suspend: true` 暂停。

**面试加分点**：
- Job 的 Pod 不会自动删除，需要 `ttlSecondsAfterFinished` 或手动清理，否则 etcd 对象膨胀。
- 需要「保证恰好一次」的场景不能只靠 Job，要有幂等设计 + 去重表。
- 大批量定时任务可考虑 KEDA + 事件驱动，或专门的 workflow 引擎（Argo Workflows）。

### 6. Service 有哪几种类型？

**要点**：ClusterIP / NodePort / LoadBalancer / ExternalName，外加 Headless。

**展开**：
- **ClusterIP**：分配一个虚拟 IP（VIP），仅在集群内可达。`kube-proxy` 在每个节点写规则把 VIP DNAT 到 Pod IP。
- **NodePort**：在 ClusterIP 之上，再在每台节点上监听 `30000–32767` 的端口；`nodePort` 可手动指定。适合本地/裸金属。
- **LoadBalancer**：在 NodePort 之上，由 CCM 向云厂商申请 LB，回写 `status.loadBalancer.ingress`。裸金属需要 MetalLB 之类的实现。
- **ExternalName**：不分配 VIP，DNS 返回 CNAME 到外部域名，用于把外部依赖伪装成集群内 Service。
- **Headless（clusterIP: None）**：不分配 VIP，DNS 直接返回所有就绪 Pod 的 A/AAAA 记录；客户端自己负载均衡。StatefulSet 和「需要访问所有副本」的场景（如 gRPC 客户端侧负载均衡、Redis cluster）用它。

**面试加分点**：
- ClusterIP 不是「一台机器」，而是 iptables/IPVS 规则，所以 `ping VIP` 不通是正常的（ICMP 不会被转发）。
- `TYPE=LoadBalancer` 的 Service 一定同时有 ClusterIP 和 NodePort（除非用 `allocateLoadBalancerNodePorts: false`）。
- 大规模下 NodePort 端口数量有限（约 2768 个），端口规划要提前做。

### 7. Service 的负载均衡是怎么实现的？

**要点**：控制面把 Service 编成 Endpoints/EndpointSlice，数据面（kube-proxy 或 eBPF）编成转发规则。

**展开**：
- **控制面**：apiserver 存 Service；EndpointSlice controller 根据 selector 找到 Ready 的 Pod，写入 EndpointSlice。
- **iptables 模式**：为每个 Service 生成 `KUBE-SERVICES` 链，DNAT 到随机挑选的后端；规则数量 O(Service × Endpoint)，更新时全量重写，规模大时慢。
- **IPVS 模式**：在内核用哈希表管理 VIP→RS，支持 rr/lc/sh/sh/sed/nq 等调度算法，规则更新 O(1)，大规模更好；但需要内核模块，且 conntrack 压力仍在。
- **nftables 模式**（1.31+ GA）：替代 iptables，规则集更小、更新更快。
- **eBPF（Cilium）**：直接把 Service 映射写进 eBPF map，用 `XDP/TC` 提前处理，可绕过 kube-proxy，还能做七层策略与拓扑感知。
- 会话保持（`ClientIP`）靠 conntrack，不是应用层会话。

**面试加分点**：
- 默认负载均衡在 iptables 下是「概率均分」，后端数量少时可能不均匀；IPVS 的 rr 更均匀。
- DNAT 后源 IP 会变（除非 `externalTrafficPolicy: Local` + `preserve client IP`），这常导致应用日志里全是节点 IP。
- 大规模集群里首包延迟/连接复用对负载均衡影响很大，gRPC 长连接会导致负载严重倾斜 —— 需要客户端侧 LB（headless + 客户端 RR）。

### 8. Ingress 是什么？和 Service 的关系？

**要点**：Ingress 是声明式的七层路由规则，由 Ingress Controller 实现；它把域名/路径映射到 Service。

**展开**：
- Ingress 资源本身不干活，是「配置对象」；真正监听 80/443 的是 Ingress Controller（nginx-ingress、Traefik、HAProxy、Envoy Gateway、云 LB Controller）。
- 常见能力：基于 Host、Path 的路由，TLS 终止（`tls` 字段引用 Secret），路径重写（靠 Controller 注解/CRD），灰度（nginx `canary` 注解），限流与超时（注解）。
- 请求链路：用户 → 云 LB / NodePort → Ingress Controller Pod → 根据规则 → Service → Pod。
- 注解是「厂商特定」的，可移植性差，这是 Gateway API 提出的重要原因。

**面试加分点**：
- Ingress Controller 自身是 Pod，需要高可用（多副本 + PDB + 反亲和）；它的故障会导致全站入口不可用。
- 与 Service 的关系是「被引用」：Ingress 里写 `service.name` 和 `service.port`，如果 Service 改名或端口不匹配，会出现 503。
- Ingress 不支持 TCP/UDP（部分实现用 `tcp-services` ConfigMap 绕过），原生四层用 `Service type=LoadBalancer` 或 Gateway API 的 `TCPRoute`。

### 9. ConfigMap 与 Secret 的区别？

**要点**：ConfigMap 存普通配置，Secret 存敏感数据，但两者都只是「一种存储对象」，不是加密保险箱。

**展开**：
- ConfigMap：`data`（UTF-8 字符串）/`binaryData`（base64），可挂载为文件或注入环境变量；`immutable: true` 可提升性能并防止误改（但改内容需重建）。
- Secret：`data` 是 base64；`stringData` 写入时自动编码。类型有 `Opaque`、`kubernetes.io/tls`、`kubernetes.io/dockerconfigjson`、`kubernetes.io/service-account-token`，类型决定校验规则。
- 挂载为卷的 Secret 默认 `mode: 0644`（需 `defaultMode` 收紧），以 `tmpfs` 形式挂载，不落盘。
- etcd 静态加密：`EncryptionConfiguration` 指定 `aescbc`/`aesgcm`/`secretbox`/KMS。**默认不加密**，只在 etcd 磁盘上静止加密，传输仍靠 TLS。

**面试加分点**：
- base64 只是编码，不是加密；拿到 Secret 的 `kubectl get -o yaml` 就等于拿到明文，所以 RBAC 必须收窄。
- 环境变量注入的 Secret **不会热更新**；卷挂载的会（kubelet 定期同步，约 1 分钟左右），但应用需要自己重新读取文件。
- 生产建议：External Secrets Operator / Vault Agent Injector / Secrets Store CSI Driver，让 Secret 只在内存或临时卷中出现，不长驻 etcd。

### 10. Namespace 的作用是什么？它是隔离边界吗？

**要点**：是「名称作用域 + 资源配额 + RBAC 边界」，但**默认不是网络和内核级隔离**。

**展开**：
- 提供：资源名唯一性、`kubectl -n` 作用域、ResourceQuota/LimitRange、Role/RoleBinding、NetworkPolicy 的默认作用域、Pod Security Admission 的标签作用域。
- 不提供：网络隔离（需 NetworkPolicy）、CPU/内存的硬隔离（节点共享内核）、主机 PID/IPC 隔离、存储隔离、内核参数隔离。
- 特殊命名空间：`kube-system`（控制面组件）、`kube-public`（可读的公共配置，如 `cluster-info`）、`kube-node-lease`（节点心跳 Lease）、`default`。
- 跨命名空间的引用：Service 用 FQDN，PVC 不能跨 namespace 引用，`RoleBinding` 可以引用其他 ns 的 ServiceAccount（`subjects[].namespace`）。

**面试加分点**：
- 「Namespace 能当租户边界吗？」——软多租户可以（RBAC + Quota + NetworkPolicy + PSA），硬多租户不行，需要独立集群或 vCluster/Kata 这类更强隔离。
- `kubectl delete ns` 是异步的，会卡在 Terminating 直到所有资源 GC 完；长 finalizer 是常见卡点。
- 大量 namespace（数千）会增加 apiserver 的 watch 与 controller 负担，需评估。

### 11. Label 与 Annotation 的区别？

**要点**：Label 用于**选择**（Selector 可查询），Annotation 用于**附加元数据**（不可选择）。

**展开**：
- Label：`key=value`，长度受限（63 字符）、字符集受限；支持 `app`, `version`, `env`, `component` 等惯例；`kubectl get -l`、Service selector、NetworkPolicy podSelector、PVC selector 都基于它。
- Annotation：可存任意大小文本（etcd 单对象总量约 1.5MB 限制），常见用途：Ingress Controller 配置、变更原因（`kubernetes.io/change-cause`）、构建信息、`kubectl.kubernetes.io/last-applied-configuration`、`deployment.kubernetes.io/revision`。
- 两者都用 `key`（可带 DNS 前缀 `example.com/foo`）约束；Label 还有个 `node.kubernetes.io/` 等系统前缀。

**面试加分点**：
- 用 Label 做版本选择时，注意 `version` 会随发布变化，Service selector 里通常**不应该**带 version（否则发布期间会丢后端）。
- 用 Annotation 存大 JSON 会显著增加 etcd 写入量与 watch 流量，大对象应存 ConfigMap/对象存储。
- 不可变 Label 场景：Pod 的 `nodeName`、拓扑标签由系统维护，不要手动改。

### 12. 什么是 OwnerReference 和垃圾回收？

**要点**：子对象通过 `ownerReferences` 声明父对象，父对象删除时由 GC 控制器级联清理。

**展开**：
- `ownerReferences` 包含 `apiVersion`、`kind`、`name`、`uid`、`controller`、`blockOwnerDeletion`。`uid` 保证「同名重建的父对象」不会误删新子对象。
- 仅支持**同 namespace** 或**集群级** owner；跨 namespace 的 owner 是非法的（会打印 warning 并不生效）。
- 传播策略：`Background`（默认，先删父再异步删子）、`Foreground`（父先加 finalizer，等子删完再删父）、`Orphan`（保留子对象，去掉 ownerReference）。
- GC 控制器（在 kube-controller-manager 里）定期扫描，把「owner 不存在」的对象删除；`--enable-garbage-collector` 必须为 true（kubeadm 默认开）。

**面试加分点**：
- 常用技巧：用 `kubectl tree` 查看对象的 owner 链条，排查「删了 Deployment 但 Pod 还在」。
- 删除 Deployment 时 `--cascade=orphan` 会保留 ReplicaSet，用于「迁移管理权」到另一个控制器。
- 误删的根源通常是 `uid` 不匹配或 finalizer 阻塞。

### 13. Finalizer 是什么？

**要点**：一个「删除前置条件列表」，只有所有 finalizer 都被移除，对象才会真正从 etcd 删除。

**展开**：
- 删除流程：`DELETE` → apiserver 设置 `metadata.deletionTimestamp` 与 `gracePeriodSeconds` → 对象进入 Terminating，仍在 etcd 中但只读 → 各控制器执行清理并 `patch` 去掉自己的 finalizer → finalizer 为空后 apiserver 物理删除。
- 典型用者：PV 保护（`kubernetes.io/pv-protection`，防止 PVC 还绑着时就删 PV）、`foregroundDeletion`、CCM 清理云 LB、Operator 清理外部资源、Argo CD 的 `resources-finalizer.argocd.argoproj.io`。
- 卡住的处理：`kubectl patch <res> -p '{"metadata":{"finalizers":[]}}' --type=merge` 强制清空（**危险**，会泄漏外部资源）；或修复对应控制器。

**面试加分点**：
- 「Namespace 一直 Terminating」经常就是内部某个对象有 finalizer 没清掉。
- finalizer 与 ownerReference 的区别：ownerRef 决定「谁删我」，finalizer 决定「我什么时候能被删」。
- 自定义控制器实现 finalizer 时必须幂等，且要考虑崩溃后重入。

### 14. CRD 与 Operator 的关系？

**要点**：CRD 只是「新增一种 API 类型」，Operator 是「CRD + 控制器 + 运维知识」。

**展开**：
- CRD 定义 schema（`OpenAPIV3Schema`）、版本（`versions[]`，可多版本共存）、作用域（Namespaced/Cluster）、printer columns、subresources（`status`/`scale`）、`conversion` webhook（多版本转换）。
- 控制器用 client-go/Kubebuilder 的 informer watch CR，执行 reconcile，并把结果写回 `status`（`spec` 是期望，`status` 是实际）。
- Operator 解决的问题：数据库主从切换、备份、升级编排、扩缩容的领域知识——这些 K8s 原生控制器不懂。
- 常见框架：Kubebuilder（controller-runtime）、Operator SDK、KubeOps（.NET）、Java Operator SDK。
- Operator 的治理能力：`OLM`（Operator Lifecycle Manager）管理安装、升级和权限（`ClusterServiceVersion`）。

**面试加分点**：
- 「Operator 挂了会怎样？」CR 还在、数据还在，但无法继续调和；通常用 leader election + 多副本保证 HA。
- CRD 的坑：改了 OpenAPI 校验后旧对象不会被校验；删除 CRD 会级联删掉所有 CR（默认），需谨慎。
- 同类的「声明式控制器」还有 aggregated API Server（更难维护，用 APIService 注册），CRD 是低成本的扩展方式。

### 15. Kustomize 与 Helm 的区别？

**要点**：Helm 是「模板 + 打包 + 发布管理」，Kustomize 是「无模板的补丁叠加」。

**展开**：
- Helm：`Chart.yaml` + `values.yaml` + `templates/`（Go template + sprig 函数）+ `helm install/upgrade/rollback`。有 release 概念（Secret 中存状态），有仓库（OCI/HTTP），有 hooks（pre-install/post-install）。
- Kustomize：`kustomization.yaml` 声明 resources、patches、images、configMapGenerator、namespace、commonLabels；`base/` + `overlays/dev|prod` 叠加。不需要模板语言，保持 YAML 原形。
- 集成：`kubectl apply -k`、`kustomize build` 可直接输出，Argo CD/Flux 都原生支持。
- 常见组合：第三方软件用 Helm Chart 分发，自己的环境差异用 Kustomize overlay；或 Helm 渲染后再 Kustomize 打补丁（Argo CD 的 `kustomize.buildOptions` / Helm + Kustomize 混用需要小心）。

**面试加分点**：
- Helm 的 `values` 合并是「覆盖式」的，大列表很难局部改；Kustomize 的 patch 能精确到字段，更适合多环境。
- Helm 的 release 状态在集群里，`helm upgrade` 失败后可能处于 `pending`，需要 `--atomic`/`--cleanup-on-fail` 或手动 `helm rollback`。
- 两者都不是 GitOps 本身，真正的 GitOps 还需要一个持续对账的 Agent。

---

## 二、架构与组件（16–30）

### 16. 描述 Kubernetes 的整体架构。kube-controller-manager 和 cloud-controller-manager 有什么区别？

**要点**：控制面（apiserver、etcd、scheduler、KCM、CCM）+ 数据面（kubelet、kube-proxy、运行时），通过 CRI/CNI/CSI 解耦。

**展开**：
- 控制平面：
  - **kube-apiserver**：唯一与 etcd 对话的组件，处理认证、授权、准入、校验、持久化，提供 REST API 和 watch。
  - **etcd**：强一致 KV，存全部集群状态。
  - **kube-scheduler**：决定 Pod 去哪个节点，只写 `bind`。
  - **kube-controller-manager（KCM）**：一堆控制器共用一个二进制，watch + reconcile 集群内对象。
  - **cloud-controller-manager（CCM）**：可选，把集群对象映射到云资源。
- 数据平面：
  - **kubelet**：节点代理，管理 Pod 生命周期、探针、卷、cgroup、上报状态。
  - **kube-proxy**：Service 转发规则（或用 Cilium 的 eBPF 替代）。
  - **容器运行时**：containerd / CRI-O，通过 CRI gRPC 交互。
- 三类接口：CRI（运行时）、CNI（网络）、CSI（存储），让 K8s 不绑定到具体实现。

**KCM 与 CCM 的区别**：

| 维度 | kube-controller-manager（KCM） | cloud-controller-manager（CCM） |
| --- | --- | --- |
| 定位 | 集群内对象的「状态调和」 | 「云适配层」，连接 K8s 对象与云资源 |
| 是否必需 | 必需，核心组件 | 可选，仅在公有云/有云平台的集群中存在 |
| 依赖 | 只需 apiserver/etcd | 云厂商 SDK、凭据（ServiceAccount + IAM 等） |
| 典型控制器 | Deployment、ReplicaSet、StatefulSet、DaemonSet、Job、Node（健康/驱逐）、EndpointSlice、Namespace、ServiceAccount、PV 绑定、HPA、GC | Node（打 ProviderID/zone label、判断节点是否已被销毁）、Service（创建/更新/删除 LB、安全组、回写地址）、Route（更新 VPC 路由） |
| 作用对象 | Pod、工作负载、Namespace、PV/PVC 等 | VM、LB、EIP、路由表等云资源 |
| 典型故障现象 | Pod 不进 Endpoints、ReplicaSet 不扩缩、节点被误判 NotReady | LB 一直 Pending / 泄漏、节点缺 zone 标签导致拓扑调度失效、跨节点路由不通 |

**关键点**：
1. **拆分原因**：早期云逻辑（AWS/GCE 的节点、LB、路由）直接写在 KCM 里，导致 core 与云厂商强耦合、发版节奏被绑定。1.6 引入 `--cloud-provider=external`，把云逻辑移到独立进程，由云厂商维护。
2. **职责边界**：KCM 管「集群内部应该是什么样」，CCM 管「集群对象 ↔ 云资源」的映射。`type: LoadBalancer` 的 Service，KCM 只维护 Endpoints，真正创建 LB 并回写 `status.loadBalancer.ingress` 的是 CCM。
3. **Node 生命周期被拆成两半**：KCM 的 node controller 负责心跳超时判定与 Pod 驱逐；CCM 的 node controller 负责云信息标签、ProviderID、判断节点是否已在云上被销毁（然后删除 Node 对象）。
4. **完整链路**：用户建 Service → apiserver 持久化 → KCM 更新 Endpoints → CCM 的 service controller 调云 API 建 LB/监听器/后端组 → 回写 status → 云 LB 健康检查通过，流量可达。

**面试加分点**：
- 「CCM 挂了会怎样？」Service 一直 `<pending>`、节点缺云标签（拓扑调度失效）、路由不更新；但 Pod 调度、扩缩容等核心能力不受影响。
- 「为什么 kubelet 也要 `--cloud-provider=external`？」节点初始化交给 CCM，避免 kubelet 启动时阻塞在云 API 上，云 API 抖动不应拖垮节点注册。

### 17. kube-apiserver 做了什么？

**要点**：集群的唯一 API 入口，负责认证 → 授权 → 准入 → 校验 → 持久化，并提供 watch 机制。

**展开**：
- **请求链路**（顺序很重要）：
  1. **认证**（Authentication）：客户端证书、Bearer Token（ServiceAccount JWT）、OIDC、Webhook。失败返回 401。
  2. **授权**（Authorization）：RBAC / Node / Webhook。失败返回 403。
  3. **准入**（Admission）：Mutating（改对象，如 sidecar 注入、默认值）→ 对象校验（schema、immutable 字段）→ Validating（拒绝违规）。
  4. **持久化**：编码为 protobuf 写入 etcd，版本号（`resourceVersion`）递增。
  5. **通知**：通过 watch 广播事件给控制器的 informer。
- **能力**：REST API（`/api`、`/apis`、`/apis/<group>/<version>`）、watch（`?watch=true`）、分页（`limit`/`continue`）、`fieldSelector`/`labelSelector`、`resourceVersion` 一致性读、Patch（JSON Patch / Merge Patch / Strategic Merge / Server-Side Apply）。
- **高可用**：无状态，多副本 + LB，etc比 大多数读请求走 watch cache（`--watch-cache`），减少 etcd 压力。写请求必须经 etcd Quorum。
- **限流**：API Priority and Fairness（APF）按 FlowSchema/PriorityLevelConfiguration 分优先级排队，防止一个控制器打爆 apiserver。

**面试加分点**：
- 「为什么 CRD 对象能直接 kubectl get？」因为 apiserver 有通用存储层（`etcd3` storage）和动态 schema，CRD 只是注册进去。
- 「apiserver 可以水平扩展吗？」可以，但写放大与 watch 会给 etcd 压力；etcd 才是瓶颈。
- `resourceVersion` 用于乐观并发：Update 时必须带上读到的 version，冲突返回 409。

### 18. etcd 在集群中的作用与运维要点？

**要点**：强一致的 KV 存储，是集群唯一真相；性能与磁盘 IO 直接决定控制面可用性。

**展开**：
- 使用 Raft 共识：3 节点容忍 1 节点故障，5 节点容忍 2；**偶数节点没有意义**（不能提升容错）。
- 写入路径：propose → leader 写 WAL（fsync）→ 复制到 follower → 多数派确认 → 提交 → apply 到内存中的 MVCC 存储。**fsync 延迟是最大瓶颈**，必须用 SSD。
- 默认值/限制：单请求 1.5 MiB（`--max-request-bytes`），DB 大小建议 < 8 GiB（`--quota-backend-bytes`），超过会进入 `NOSPACE` 并拒绝写。
- 日常运维：`etcdctl endpoint health/status`、`etcdctl defrag`（碎片整理，会阻塞，应逐节点执行）、`compact`（历史版本压缩，apiserver 会触发，但要监控 revision）、快照备份 `etcdctl snapshot save` + 定期恢复演练。
- 关键指标：`etcd_server_leader_changes_seen_total`、`etcd_disk_wal_fsync_duration_seconds`、`etcd_server_has_leader`、`etcd_mvcc_db_total_size_in_bytes`、`etcd_network_peer_round_trip_time_seconds`。

**面试加分点**：
- 「etcd 磁盘为什么必须独立？」容器镜像、日志写同一块盘会导致 fsync 抖动，进而导致 apiserver 超时甚至 Leader 切换。
- 恢复：`etcdctl snapshot restore` 会生成新集群（新成员 ID、新 revision），**必须停掉所有 apiserver**，恢复后其他组件重连。
- 想减少 etcd 压力：减少频繁更新的大对象、用 Lease 代替频繁心跳、开启 APF、合理设置 `--event-ttl`（事件过多很耗 etcd）。

### 19. kube-scheduler 的调度流程？

**要点**：`Filter`（可行节点）→ `Score`（打分）→ `Reserve/Permit`（预留）→ `Bind`（绑定）；调度框架（Scheduling Framework）把这些切成可插拔的扩展点。

**展开**：
- **Filtering**：NodeUnschedulable、NodeResourcesFit（资源够不够）、TaintToleration、NodeAffinity、InterPodAffinity、TopologySpread、VolumeBinding（卷能否挂到该节点）、PodTopologySpread 等。
- **Scoring**：NodeResourcesBalancedAllocation、NodeResourcesLeastAllocated（默认）、InterPodAffinity、ImageLocality（镜像已在节点上加权）、TaintToleration、PodTopologySpread。
- **Reserve / Permit / Bind**：Reserve 阶段占住资源，Permit 支持「等待其他 Pod」（如 gang scheduling），Bind 阶段写 `spec.nodeName`（新版是 PodScheduling context，为了同时绑定多个 Pod）。
- **抢占**：高优先级 Pod 若无处可去，可驱逐低优先级 Pod（`priorityClassName` + `preemptionPolicy`）。
- **配置**：`KubeSchedulerConfiguration` 的 profiles、plugins、`percentageOfNodesToScore`（默认自适应，大集群默认更小）、`nodeAffinity` 的 `nodeSelector` 快路径。

**面试加分点**：
- 调度结果 = 一个绑定事件，**不是**「启动容器」；启动由 kubelet 完成。
- 「调度延迟高」通常来自大量 Pod 同时创建 + 亲和性过滤慢 + 卷绑定等待；可看 `scheduler_scheduling_attempt_duration_seconds`。
- 常见 Pending 原因用 `kubectl describe pod` 的 `Events` 就能看到 `FailedScheduling: 0/5 nodes are available: 3 Insufficient cpu, 2 node(s) had untolerated taint`。

### 20. 常见调度策略有哪些？

**要点**：从「硬约束」到「软偏好」分层选择。

**展开**：
- **节点选择**：`nodeSelector`（精确 label 匹配）、`nodeAffinity`（`requiredDuringSchedulingIgnoredDuringExecution` 硬性 / `preferred...` 软性，支持 In/NotIn/Exists/Gt/Lt 运算）、`nodeName`（跳过调度器，直连）。
- **Pod 亲和/反亲和**：`podAffinity`（和某类 Pod 同拓扑）、`podAntiAffinity`（如同一服务副本打散）、`topologyKey` 指定域（`kubernetes.io/hostname`、`topology.kubernetes.io/zone`）。注意硬反亲和在大规模下会导致「副本数超过节点数就 Pending」。
- **拓扑分布**：`topologySpreadConstraints`（`maxSkew`、`whenUnsatisfiable: DoNotSchedule/ ScheduleAnyway`、`topologyKey`），是「多可用区均匀分布」的推荐做法，比硬反亲和更灵活。
- **污点与容忍**：节点 taint 排斥 Pod，Pod toleration 放行。
- **优先级与抢占**：`priorityClassName` 高优先级可抢占低优先级 Pod。
- **资源打分策略**：`LeastAllocated`（默认，倾向于丢到空闲节点）、`MostAllocated`（装箱，省钱）、`BalancedAllocation`（CPU/内存均衡）。
- **其他**：`dnsPolicy`/`hostNetwork` 影响调度无意义、`schedulerName` 可指定自定义调度器（如 Volcano 的 gang/批调度）。

**面试加分点**：
- 反亲和的常见坑：Deployment 副本数与节点数/可用区数不匹配，导致 Pending；用 `topologySpreadConstraints` + `whenUnsatisfiable: ScheduleAnyway` 更稳。
- `requiredDuringSchedulingIgnoredDuringExecution` 的「IgnoredDuringExecution」意味着节点 label 变化后**不会驱逐**已运行的 Pod。
- GPU/专用节点用 taint + toleration，配合 node affinity 保证只被需要的人用。

### 21. Taint 与 Toleration 怎么用？

**要点**：Taint 是节点给 Pod 的「拒绝标签」，Toleration 是 Pod 的「豁免声明」；两者必须匹配才能被调度。

**展开**：
- 格式：`key=value:Effect`，Effect 有：
  - `NoSchedule`：新 Pod 不调度，已有 Pod 不受影响。
  - `PreferNoSchedule`：软性，尽量不调度。
  - `NoExecute`：既阻止新 Pod，又**驱逐**不容忍的已有 Pod；可配 `tolerationSeconds` 宽限期。
- 容忍写法：完全匹配 `key=value:Effect`，或 `operator: Exists`（忽略 value），`effect` 可省略表示匹配所有 effect。
- 系统污点：`node.kubernetes.io/not-ready`、`unreachable`（kubelet 心跳丢失时自动打，NoExecute，默认 300s 后驱逐）、`memory-pressure`、`disk-pressure`、`pid-pressure`、`unschedulable`。
- 常见用途：GPU 节点专用（`nvidia.com/gpu=present:NoSchedule`）、控制面隔离（`node-role.kubernetes.io/control-plane:NoSchedule`）、节点维护（`kubectl taint nodes n1 maintenance=true:NoSchedule`）。

**面试加分点**：
- 「为什么 DaemonSet 要考虑容忍度？」不写容忍，节点 NotReady 时 DaemonSet Pod 也会被驱逐，导致日志/网络代理消失；通常要容忍 `not-ready`、`unreachable`、`disk-pressure` 等。
- Taint 只影响调度，不影响已运行 Pod（除了 NoExecute）；真正「赶走」要 drain/cordon。
- 多副本服务跨 zone 用 topologySpread 而不是 taint：taint 是「隔离」，不是「分布」。

### 22. kube-controller-manager 里的控制器举例？

**要点**：一组「watch + reconcile」循环，共享一个进程，各自有 leader election。

**展开**：
- **工作负载**：Deployment（管 RS 与 rollout）、ReplicaSet（管 Pod 数量）、StatefulSet（有序启停与 PVC）、DaemonSet（每节点一个）、Job/CronJob（批任务）。
- **服务与网络**：EndpointSlice（根据 Service selector 生成后端）、Node（节点健康与驱逐）、ServiceAccount（创建默认 SA）、Namespace（删除、注入）、Service CIDR 分配。
- **存储**：PersistentVolume（绑定/回收）、PersistentVolumeClaim（保护）、Attach/Detach（由 kubelet 与 CSI 协作）。
- **其他**：GC（ownerReference 清理）、TTL（TTLAfterFinished）、HPAController、PodGC（清理已完成/失败 Pod）、ResourceQuota、CSR（证书签发）、Token（SA token 清理）、Bootstrap（kubeadm 加入节点）。

**面试加分点**：
- KCM 是单进程多 controller，可用 `--controllers=*,bootstrapsigner,tokencleaner` 选择性开关；规模大时可拆成多个 KCM 实例或使用 controller-runtime 独立部署。
- 所有控制器通常启用 leader election（`LeaderElection` Lease 对象），只有 leader 执行 reconcile，避免重复操作。
- 「为什么删除 Deployment 后 RS 没立刻消失？」——GC 异步，且可能有 finalizer。

### 23. kubelet 负责什么？

**要点**：节点上的「Pod 代理」，把 API Server 声明的 Pod 变成真实进程。

**展开**：
- 主要工作：
  1. 注册节点（Node 对象 + Lease 心跳）。
  2. watch 分配给本节点的 Pod（`spec.nodeName == 本节点` 或 Static Pod）。
  3. 通过 CRI 拉镜像、创建/删除容器，管理 cgroup。
  4. 执行探针（liveness/readiness/startup，由 kubelet 调用容器内 exec/http/tcp）。
  5. 挂载卷：调用 CSI node driver 的 `NodeStageVolume` / `NodePublishVolume`。
  6. 上报状态（Pod status、Node conditions、容量、镜像）。
  7. 节点压力驱逐（`--eviction-hard`，如 `memory.available<100Mi`）。
  8. 管理证书轮换（`RotateKubeletClientCertificate`）。
- 不负责：调度、控制副本数、滚动更新策略（那些是控制面的事）。

**面试加分点**：
- 「kubelet 挂了 Pod 还会跑吗？」已启动的静态 Pod 与容器继续运行（容器不受 kubelet 控制），但状态无法上报、探针不执行、无法重建；节点最终被标记 NotReady。
- 「kubelet 与 apiserver 断连后怎么办？」kubelet 有本地缓存，重连后 reconcile；Pod 会被节点控制器按 `node-monitor-grace-period` 标记 NotReady 并驱逐（可配置容忍度）。
- 常用排障：`journalctl -u kubelet`、`/var/lib/kubelet/config.yaml`、`kubectl get --raw "/api/v1/nodes/<n>/proxy/healthz"`。

### 24. kube-proxy 有哪几种模式？

**要点**：userspace（已废弃）→ iptables → IPVS → nftables；大规模更推荐 IPVS/eBPF/nftables。

**展开**：
- **iptables**：每节点全量规则链，Service 数量级到几千时规则爆炸、更新慢（每次改动要重写链）；随机概率转发。
- **IPVS**：内核 LVS，哈希表 O(1) 查找与更新；支持 rr/lc/sh/sed/nq/... 调度；需要 `ip_vs` 内核模块；conntrack 依然是瓶颈。
- **nftables**（1.29 alpha，1.31 GA）：更小的规则集与增量更新，逐步替代 iptables。
- **eBPF（Cilium）**：替换 kube-proxy（`kubeProxyReplacement: true`），在内核直接做 DNAT，性能更好，还能做拓扑感知、七层策略。
- `--mode` 参数在 kube-proxy 启动时指定；集群内要求所有节点一致。

**面试加分点**：
- IPVS 下 Service 的 VIP 需要 `kube-ipvs0` 虚拟网卡来「持有」地址，所以 `ip addr` 能看到。
- Localhost 访问 ClusterIP 时，iptables 的 DNAT 在 OUTPUT 链也会生效，但源地址保持本机。
- 故障现象：Service 存在但 `curl` 不通、Pod 正常——先看 `ipvsadm -Ln` / `iptables -t nat -L KUBE-SERVICES`。

### 24.1 kube-proxy 是部署在每个节点上的吗？

**要点**：是。每个节点都要有，因为它是「本节点流量走向 Service 的规则维护者」。

**展开**：
- Service 的 ClusterIP 是虚拟地址，集群里没有任何主机真正拥有它；它能被访问，靠的是**每个节点自己**的转发规则。节点 1 上的 Pod 访问 Service，走的是节点 1 的网络栈，所以节点 1 必须有规则。
- 典型部署形态是 **DaemonSet**（`kube-system/kube-proxy`），使用 `hostNetwork: true`，并挂载 `xtables`、`conntrack`、`nsenter` 等宿主工具，因为它要写宿主的 iptables/IPVS 规则。
- **kube-proxy 本身不转发流量**，只「写规则」；真正转发的是内核。所以 kube-proxy 挂掉后，已有规则继续生效，只是不再随 Service/Endpoint 变化更新。

**例外与替代**：
- 控制面节点：kubeadm 默认也跑；托管集群的控制面节点不在你的节点池里，自然没有。
- **eBPF 替代**：Cilium 的 `kubeProxyReplacement: true` 不部署 kube-proxy，改由每节点上的 Cilium Agent 用 eBPF 完成转发——**仍然是每节点一个**。
- NodeLocal DNSCache 之类的组件也是每节点一个，但它们不替代 kube-proxy。

### 24.2 DNAT 是什么？

**要点**：Destination NAT，在转发过程中把数据包的**目标 IP/端口**改写成另一个。

**展开**：
- 访问 Service 时：Pod 发出 `dst = 10.96.0.10:80`（ClusterIP，虚拟地址）→ 节点内核的 iptables/IPVS 规则做 DNAT → 实际发出时 `dst = 10.244.1.7:8080`（真实 Pod IP）。
- **DNAT vs SNAT**：DNAT 改目标（要去哪），SNAT 改源（从哪来）；两者常同时使用——DNAT 让包去对地方，SNAT 让回包能原路返回。
- 只在**连接的第一个包**做，后续靠 conntrack 复用映射；所以 `conntrack` 表满时会出现「新连接建不上」的诡异问题。
- 对应用**完全透明**：应用看到的目标还是 ClusterIP，响应由内核反向改回。
- 只处理 **TCP/UDP，不处理 ICMP**——这就是为什么 `ping` 一个 ClusterIP 通常不通，测通不通要用 `curl`/`nc`。

**面试加分点**：
- 「Service 是负载均衡器吗？」严格说不是，它是**一个虚拟 IP + 一组 DNAT 规则**，负载均衡来自「从后端列表里按规则挑一个」。
- 不同模式的实现：iptables 用 nat 表规则、IPVS 用 `kube-ipvs0` + LVS、eBPF 直接在程序里改写目标地址。

### 24.3 iptables 模式与 IPVS 模式的区别是什么？

**要点**：iptables 是「规则链表逐条匹配」，IPVS 是「哈希表直接查」，本质是**数据结构**的差异。

**展开**：

| 维度 | iptables | IPVS |
| --- | --- | --- |
| 底层 | netfilter nat 表 | 内核 LVS 模块 |
| 查找 | 线性遍历规则链，O(N) | 哈希表，O(1) |
| 扩容性 | 每个 Service/Endpoint 一条规则，线性增长 | 一条记录对应一组后端 |
| 更新 | 每次变更**全量重写**规则，规模大时性能抖动明显 | 增量更新单条记录 |
| 调度算法 | 概率均分（随机） | rr / lc / wrr / sh / sed / nq |
| 大集群 | Service 上千后明显退化 | 更稳 |
| 依赖 | 无 | 需要 `ip_vs` 内核模块 |

- **最关键的是更新成本**：3000 个 Service × 10 个后端 ≈ 3 万条 iptables 规则，任何一个 Pod 就绪变化都可能触发全量重写；IPVS 只改受影响的一条。
- IPVS 模式下节点会多出 `kube-ipvs0` 虚拟网卡，所有 VIP 挂在其上（`ip addr` 可见）。

**常见误解**：
- ❌「IPVS 就不需要 iptables 了」——仍需要，NodePort 的 DNAT、hairpin 流量等边缘规则还在 iptables 里，只是「以 IPVS 为主、iptables 为辅」。
- ❌「IPVS 一定更快」——匹配阶段更快，但 **conntrack 仍是共同瓶颈**；小集群感受不到差异。
- ❌「换模式能解决性能问题」——大规模真正的瓶颈常在 etcd 写入、apiserver 压力与 conntrack 表。

**选型**：小中规模用 iptables（最省心）；数千 Service 或需要会话保持/特定算法用 IPVS；新集群关注 nftables 或 Cilium eBPF。


### 25. 容器运行时接口 CRI 是什么？

**要点**：kubelet 与容器运行时之间的 gRPC 契约，解耦运行时的实现。

**展开**：
- 两套服务：`RuntimeService`（创建/启停/删除容器、exec、attach、端口转发、镜像拉取）与 `ImageService`（镜像列表、删除、查询）。
- 主流实现：containerd（最广泛）、CRI-O（OpenShift 系）、Docker（dockershim 在 1.24 移除）、轻量/沙箱类（Kata、gVisor、runsc 集成）。
- 运行时之上的分层：CRI → containerd → shim（`containerd-shim-runc-v2`）→ OCI runtime（runc）→ OCI bundle（rootfs + config.json）。
- CRI 之外还有：Pod 沙箱（pause）由 `sandbox` 配置、镜像拉取凭证（`imagePullSecrets` → kubelet 传给 CRI）。

**面试加分点**：
- 「为什么移除 dockershim？」Docker 不是 CRI 实现，需要中间层，且 K8s 不想绑死；containerd 直接支持 CRI。
- 「kubelet 报 `failed to pull image` 但 docker pull 正常？」因为 kubelet 用的是 containerd 的镜像存储，凭据和 registry 配置（`/etc/containerd/config.toml` 的 `config_path`）不同。
- 运行时性能差异（cgroup v2、seccomp、rootless）是安全与性能权衡的常见追问。

### 25.1 RuntimeClass 是什么？

**要点**：一种**集群级**资源，用来声明「某个容器运行时」，并让 Pod 通过 `runtimeClassName` 指定它。一句话：**给 Pod 选运行时的插槽**。

**展开**：

三个核心字段：

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc                 # 必填：对应节点 CRI 配置里的 handler 名
overhead:                      # 可选：该运行时额外开销，调度时预留
  podFixed: { cpu: 250m, memory: 64Mi }
scheduling:                    # 可选：限定只能调度到装了该运行时的节点
  nodeSelector: { sandbox.gke.io/runtime: gvisor }
  tolerations:
    - { key: sandbox.gke.io/runtime, operator: Exists, effect: NoSchedule }
```

- **`handler`** 必须与节点 containerd 配置中定义的 runtime handler **名字完全一致**，否则报 `failed to find runtime "xxx"`。
- **`overhead`** 主要给 VM 类运行时（Kata）用：每个 Pod 多占一份 VM 开销，不声明会导致调度超卖。
- **`scheduling`** 防止 Pod 被调度到没装该运行时的节点（否则 `FailedCreatePodSandBox`）。

使用方式：

```yaml
spec:
  runtimeClassName: gvisor     # 仅这一行
```

**端到端链路**：
```
Pod 指定 runtimeClassName
  → 调度器按 scheduling 选节点，并按 overhead 预留资源
  → kubelet 查 RuntimeClass 得到 handler
  → 通过 CRI 告诉 containerd「用这个 handler」
  → containerd 启动对应 shim（如 containerd-shim-runsc-v1）
```

**与相邻概念的区别**：

| 对比 | 区别 |
| --- | --- |
| vs 镜像 | 镜像决定「跑什么」，RuntimeClass 决定「在什么环境里跑」 |
| vs 节点标签 | 标签是「选节点」，RuntimeClass 是「选运行时」，可顺带带出节点约束 |
| vs `securityContext` | 前者换底层隔离机制，后者限制进程权限；**互补而非替代** |
| vs device plugin | 后者提供资源（GPU），前者只选运行时、不提供资源 |

**面试加分点**：
- 「RuntimeClass 是命名空间级还是集群级？」**集群级**，无 namespace 字段。
- 「改了 RuntimeClass 会重启已有 Pod 吗？」不会，只对新建 Pod 生效。
- 「怎么验证真的用上了 gVisor？」`kubectl exec <pod> -- uname -r` 会显示假版本号（如 `4.4.0`），与节点内核版本不同。
- 「能否替代多租户隔离？」不能，它只解决内核级隔离这一层，还要 RBAC + NetworkPolicy + 配额配合。

### 26. CNI 是什么？常见插件有哪些？

**要点**：容器网络接口规范，负责「给 Pod 分配 IP 并把它接入网络」。

**展开**：
- 调用链：kubelet 创建 sandbox → 调用 CRI → containerd 调用 CNI 插件（`ADD`/`DEL`/`CHECK`）→ 插件配置 veth、分配 IP、设置路由/iptables。
- 配置文件在 `/etc/cni/net.d/`（`10-calico.conflist` 等），二进制在 `/opt/cni/bin`。
- 常见方案：
  - **Calico**：BGP 路由或 IPIP/VXLAN 封装，NetworkPolicy 强，性能好。
  - **Cilium**：eBPF，可替代 kube-proxy，L3–L7 策略、可观测性。
  - **Flannel**：简单 VXLAN/host-gw，**不支持 NetworkPolicy**。
  - **Antrea**：Open vSwitch，支持策略。
  - **AWS VPC CNI**：Pod 直接拿 VPC 子网 IP（ENI），无封装但 IP 消耗大。
  - **Multus**：多网卡（主网卡 + RDMA/SR-IOV 网卡）。
- 网络模型要求：Pod 之间无需 NAT 互通、节点与 Pod 无需 NAT 互通、Pod 看到的 IP 一致。

**面试加分点**：
- 「CNI 与 kube-proxy 的职责划分？」CNI 管 Pod↔Pod 的连通与 IP；kube-proxy 管 Service VIP 到 Pod 的 DNAT。
- 「网络插件升级会中断业务吗？」会，CNI DaemonSet 滚动更新时新 Pod 创建可能失败，需先滚动节点或选支持热升级的方案。
- IP 池耗尽（`Calico IPAM` / AWS ENI）是常见故障，表现是 Pod `ContainerCreating` 且事件里没有明显错误。

### 27. CSI 是什么？

**要点**：容器存储接口，把「卷供给、挂载、快照」抽象成 gRPC 服务，供 kubelet 和外部驱动对接。

**展开**：
- 组件：
  - **Controller 插件**（Deployment/StatefulSet）：`CreateVolume`/`DeleteVolume`/`ControllerPublishVolume`/`CreateSnapshot`。
  - **Node 插件**（DaemonSet）：`NodeStageVolume`/`NodePublishVolume`（把卷挂载到 Pod 的挂载点）。
  - **sidecar**：`external-provisioner`（响应 PVC 创建 PV）、`external-attacher`（Attach/Detach）、`external-resizer`（扩容）、`external-snapshotter`（快照）、`node-driver-registrar`（向 kubelet 注册）。
- 生命周期：PVC 创建 → provisioner 建卷并生成 PV → Pod 调度（`WaitForFirstConsumer` 时延迟）→ attacher 挂到节点 → kubelet 调 NodeStage/NodePublish → 容器启动 → Pod 删 → NodeUnpublish/Unstage → Detach → 回收策略（Retain/Delete）。

**面试加分点**：
- 「为什么 Pod 一直 ContainerCreating？」目录挂载失败是高频原因，看 `kubectl describe pod` 的 `FailedMount` 事件，再查 CSI node 插件日志。
- 多可用区场景必须 `volumeBindingMode: WaitForFirstConsumer`，否则卷建在 A 区、Pod 调到 B 区。
- 快照能力依赖 `VolumeSnapshotClass` + `external-snapshotter`，且不是所有驱动都支持。

### 28. 为什么要用 EndpointSlice 而不是 Endpoints？

**要点**：Endpoints 是「一个大对象」，规模大时更新放大严重；EndpointSlice 按 100 个一组切片，减少 watcher 与 apiserver 压力。

**展开**：
- `Endpoints` 的问题：一个 Service 的所有后端都在一个对象里，任何一个 Pod 的 Ready 变化都会导致整个对象重写，所有节点上的 kube-proxy 都要重新接收并重算，O(N) 放大。
- `EndpointSlice`（`discovery.k8s.io/v1`）：
  - 每个 slice 最多 100 个 endpoint（`endpointslice.kubernetes.io/max-endpoints` 注解）。
  - 带 `addressType`（IPv4/IPv6/FQDN）、`topology`（zone/region/hostname），支持拓扑感知路由。
  - 只有变动的 slice 会被更新，watch 量大幅下降。
- kube-proxy 在较新版本已只 watch EndpointSlice；`Endpoints` 仍存在主要是兼容性。

**面试加分点**：
- 「Service 有 Endpoints 但没 EndpointSlice？」检查 EndpointSlice controller 是否正常（KCM 的一个 controller）、label `kubernetes.io/service-name` 是否正确。
- 拓扑感知路由（`trafficDistribution: PreferClose`，1.31 GA）依赖 EndpointSlice 的 zone 信息，可减少跨区流量。

### 28.1 Endpoints 和 EndpointSlice 的结构差别

**要点**：Endpoints 是「一个对象装全部后端」，EndpointSlice 是「按上限切片的一组对象」。

| 维度 | Endpoints（`v1`） | EndpointSlice（`discovery.k8s.io/v1`） |
| --- | --- | --- |
| 与 Service 关联 | **同名** | label `kubernetes.io/service-name` |
| 一个 Service 对应几个对象 | **1 个** | **N 个** |
| 单对象大小 | 随后端数线性增长 | 有上限（默认 100） |
| 更新粒度 | 任一端点变化 → 整对象重写 | 只更新变化的 slice |
| 拓扑信息 | 仅 `nodeName` | `zone`、`region`、`hostname` |
| 端点状态 | 在/不在列表 | `ready` / `serving` / `terminating` 三个独立条件 |
| 地址类型 | 隐含 | 显式 `addressType`：IPv4 / IPv6 / FQDN |

**`serving` 条件的价值**：Pod 优雅退出时，Endpoints 只能表达「在」或「不在」，无法表示「别发新请求但还能处理存量」。EndpointSlice 用 `ready=false` + `serving=true` 表达了这种中间态。

### 28.2 是怎么拆分到 EndpointSlice 的

**要点**：由 **EndpointSlice controller**（kube-controller-manager 中的一个控制器）**直接从 Service + Pod 生成**，生成时按「数量上限、地址类型、端口」分片，**不按 zone 分片**。

**五条拆分规则**：

1. **数量上限（主规则）**：每个 slice 最多 100 个 endpoint（`endpointslice.kubernetes.io/max-endpoints` 注解；可用 KCM 的 `--max-endpoints-per-slice` 调整，上限 1000）。
   ```
   100 个后端 → 1 个 slice；101 个 → 2 个（100+1）；500 个 → 5 个
   ```
2. **必须按 addressType 分开**：一个 slice 内只能有一种地址类型。双栈集群下，同一 Service 会同时有 IPv4 和 IPv6 两组 slice。
3. **按端口集合分组**：端口组合不同的端点不会混在同一个 slice。
4. **装箱而非均分**：优先塞进还有空位的 slice，都满了才新建。所以填充率可能不均（如 100/100/37）。
5. **拓扑信息不参与分片**：`zone`/`region` 只是写进每个 endpoint 的字段，同一个 slice 里可以混有不同 zone 的 Pod；拓扑路由是在 slice **内部按字段过滤**。

**重要澄清**：不是「Endpoints 拆成 EndpointSlice」，而是两者都由控制器**从同一数据源独立生成**，`Endpoints` 反而是兼容产物。

**更新行为**：

| 事件 | 行为 |
| --- | --- |
| 新增 Pod | 塞进有空位的 slice，都满则新建 |
| 删除 Pod | 只从所在 slice 移除，只更新这一个对象 |
| ready → not ready | 只改该 endpoint 的 `conditions`，不删除 |
| 后端全消失 | slice 被删除或变空 |

**命名与归属**：
- 名字格式 `<service-name>-<随机后缀>`（如 `reviews-7xk2p`），**不要硬编码**。
- 归属靠 label `kubernetes.io/service-name`，管理方标记为 `endpointslice.kubernetes.io/managed-by=endpointslice-controller.k8s.io`。

**常见误解**：
- ❌「按 zone 分片」——不按 zone 分，zone 只是字段。
- ❌「平均分配」——是装箱策略，分布可能不均。
- ❌「每 100 个 Pod 一个 slice」——准确说法是「每个 slice 最多 100 个 endpoint」。
- ❌「Endpoints 已废弃」——未废弃，仍由 EndpointSlice 镜像生成以兼容老客户端，但新代码不应依赖。

**排障命令**：
```bash
kubectl get endpointslice -n <ns> -l kubernetes.io/service-name=<svc>
kubectl get endpointslice -n <ns> -l kubernetes.io/service-name=<svc> \
  -o custom-columns=NAME:.metadata.name,ADDR:.addressType,COUNT:.endpoints[*].addresses
```

### 29. 什么是 Static Pod？

**要点**：由 kubelet 直接读本地 manifest 创建的 Pod，不经过 apiserver 调度。

**展开**：
- 目录：kubelet 的 `staticPodPath`（kubeadm 默认 `/etc/kubernetes/manifests`），放入 YAML 后 kubelet 自动创建，文件删除则 Pod 删除。
- 镜像的 Pod 会以「mirror pod」形式出现在 apiserver（名字带节点名后缀，如 `etcd-node1`），只能看不能改（改会被 kubelet 覆盖）。
- 用途：引导控制面自身（apiserver、etcd、controller-manager、scheduler）、节点上的关键组件。
- 与 kubelet 重启：静态 Pod 的容器在 kubelet 重启后会重建（除非容器本身仍在运行）。

**面试加分点**：
- 「`kubectl delete` 静态 Pod 为什么删不掉？」它会立即被 kubelet 重建，因为源文件还在。
- 「静态 Pod 没有 Deployment 那样的自愈？」它们在**节点本地**自愈，但节点挂了不会迁移（这是控制面高可用的设计取舍：用多副本 + LB）。
- 与 DaemonSet 的区别：DaemonSet 由控制面调度、可跨节点，Static Pod 是节点本地、不依赖控制面。

### 30. 什么是 Aggregation Layer 和 APIService？

**要点**：把外部的自定义 API Server 注册到主 apiserver 的 URL 路径下，由主 apiserver 代理转发。

**展开**：
- 机制：创建 `APIService` 对象（`spec.group`/`version`/`service`），apiserver 把 `/apis/<group>/<version>/...` 的请求通过 Service（通常 https）转发给扩展 apiserver。
- 扩展 apiserver 需要：TLS 证书（由 `apiservice` 的 `caBundle` 校验）、实现 `TokenReview`/`SubjectAccessReview` 回调以便鉴权、提供 discovery 信息。
- 经典例子：`metrics-server`（`metrics.k8s.io`）、`custom.metrics.k8s.io`（Prometheus Adapter）、`external.metrics.k8s.io`、KEDA 相关组件。
- 与 CRD 的对比：CRD 用 K8s 通用存储，开发简单；Aggregation 可自定义存储（如 metrics 存在内存）、自定义语义，但运维复杂。

**面试加分点**：
- 「`kubectl top` 报错 `the server could not find the requested resource`」——通常 metrics-server 的 APIService 不可用，`kubectl get apiservice` 看 `v1beta1.metrics.k8s.io` 是否 Available。
- Aggregation 的可用性风险：扩展 apiserver 挂了会导致对应 API 组不可用，但不会拖垮核心 API（除非 APIService 的 `caBundle` 错误导致持续报错）。

---

## 三、工作负载与配置（31–45）

### 31. Deployment 滚动更新的过程？

**要点**：新建 ReplicaSet → 按 surge 扩容新 RS → 按 unavailable 缩容旧 RS → Ready 验证 → 直至完成。

**展开**：
- 触发：修改 Pod template 的任意字段（镜像、env、labels），Deployment 计算新 `pod-template-hash`，创建新 RS，旧 RS 保留。
- 参数：`maxSurge`（默认 25%）、`maxUnavailable`（默认 25%）、`progressDeadlineSeconds`（默认 600s，超时标记 `ProgressDeadlineExceeded`）、`minReadySeconds`（Pod Ready 后持续多久才算可用，防止「闪断」）。
- 节奏：先按 surge 扩容新 RS，等新 Pod Ready 后缩旧 RS；任何新 Pod 不 Ready 会卡住（`maxUnavailable` 耗尽时不继续）。
- 状态观察：`kubectl rollout status deploy/x`、`kubectl rollout history deploy/x`、`kubectl describe deploy` 里的 `Events` 与 `Conditions`。

**面试加分点**：
- 「滚动更新为什么会有 5xx？」readiness 未配好/new Pod 过早接流量、terminationGracePeriod 太短、preStop 缺失导致旧 Pod 被摘之前就断连接。
- `maxSurge: 0, maxUnavailable: 0` 是非法组合（必须至少一个 > 0）。
- Deployment 的 `revisionHistoryLimit` 用完后，旧 RS 会被 GC 掉，届时无法 `--to-revision` 回滚。

### 32. 如何实现灰度 / 金丝雀发布？

**要点**：用「流量比例 + 健康指标 + 自动放量」的闭环，而不是简单多套 Deployment。

**展开**：
- **按副本比例**：stable + canary 两个 Deployment，Service selector 同时选中；缺点是比例受副本数粒度限制，且新版本一旦有 bug 影响面按副本数扩散。
- **Ingress 层**：nginx `canary`/`canary-weight`/`canary-by-header` 注解；Envoy Gateway/APISIX 的 route 权重。
- **服务网格**：Istio `VirtualService` 权重、`DestinationRule` 子集、`mirror`（影子流量）、一致性哈希。
- **渐进式交付工具**：Argo Rollouts（`Rollout` CRD + `AnalysisTemplate` + `Experiment`）、Flagger（基于 mesh 的自动放量分析）、OpenFeature 特性开关。
- **数据兼容性**：金丝雀前确保 DB schema 向后兼容（expand-contract 模式），否则回滚也救不了。

**面试加分点**：
- 「如何自动判断是否继续放量？」看 SLI：错误率、P99 延迟、业务指标（下单成功率）。可用 Prometheus 查询 + Argo Rollouts 的 analysis。
- 「蓝绿和灰度区别？」蓝绿是两套完整环境切换，回滚快但资源翻倍；灰度是按比例放量，资源增量小但需要路由能力。
- 「为什么不能只靠副本比例做灰度？」因为长连接（gRPC/WebSocket）下流量不会均匀分配，比例失真。

### 32.1 蓝绿、灰度、金丝雀到底怎么区分？

**要点**：**金丝雀是灰度的一种**；蓝绿则是另一种思路——两套完整环境 + 一次性切换。

| 策略 | 一句话 | 关键特征 |
| --- | --- | --- |
| **蓝绿（Blue-Green）** | 准备两套完整环境，**一次性切换流量** | 全有或全无，回滚极快 |
| **灰度（Gray Release）** | 按**比例或规则**逐步放量，是**总称** | 大部分流量不变，小部分试新 |
| **金丝雀（Canary）** | 灰度的一种，**先给极少数流量试** + 指标判定 | 小流量试跑 → 观察 → 逐级放大 |

**关系**：`金丝雀 ⊆ 灰度`，两者不是并列关系。

#### 蓝绿

```
切换前：蓝(旧) ← 100%     绿(新) ← 0%（已部署待验证）
切换：  路由一次性切到绿
切换后：绿(新) ← 100%     蓝(旧) ← 0%（保留待回滚）
```

- **优点**：回滚最快（切回路由即可，秒级）；切换语义清晰，没有中间态。
- **缺点**：**资源翻倍**；切换瞬间**全部用户**一起受影响；长连接/有状态服务难切换。
- **最大难点是数据库**：新旧两版同时读写同一库，schema 必须**双向兼容**，这也是蓝绿很少用于有状态核心服务的原因。

#### 灰度

「按规则分流」的手段集合，分流维度包括：

| 维度 | 例子 |
| --- | --- |
| 比例 | 5% → 25% → 50% → 100% |
| 用户特征 | 用户 ID 哈希、地域、会员等级 |
| 请求特征 | Header、Cookie、设备类型 |
| 时段 | 先夜间放量 |

#### 金丝雀

强调**「小流量 + 指标判定 + 自动放量」的闭环**：

```
放量 → 采集 SLI（错误率/P99/业务指标）→ 与基线对比 → 达标继续，否则自动回滚
```

**和普通灰度的细微差别**：灰度侧重「按比例分流」这一手段，金丝雀侧重「从极小流量开始 + 指标驱动决策」这一流程。实践中两者常混用。

#### 三方对比

| 维度 | 蓝绿 | 灰度 | 金丝雀 |
| --- | --- | --- | --- |
| 切换方式 | **一次性全切** | 按比例渐进 | 极小比例 → 渐进 |
| 同时存在 | 2 套完整环境 | 2 个版本 | 2 个版本 |
| 资源开销 | **大（翻倍）** | 小 | 小 |
| 故障影响面 | 100%（切换后） | 按比例 | 极小，逐步扩大 |
| 回滚速度 | **最快**（切路由） | 快（调回比例） | 快 |
| 依赖能力 | 双环境 + 路由切换 | 流量分流 | 分流 + **指标自动化** |

#### 别忘了第四个选项：滚动更新

K8s 原生的 `RollingUpdate` **不属于上面任何一种**：它只保证「不中断」，**不控制流量比例、不做指标判定**，只是分批替换 Pod。常被误当作灰度。

#### 选型

```
风险很高、必须秒级回滚 ──▶ 蓝绿（前提：资源够、DB 兼容）
需要边观察边放量     ──▶ 金丝雀（1% → 10% → 50% → 100%）
要对比两版效果(A/B)  ──▶ 按用户分流的灰度
只是无状态常规发布   ──▶ 滚动更新即可
```

#### 常见坑

- **「金丝雀 1%」可能不准**：按副本数分流时粒度很粗（3 副本 → 33%）；长连接（gRPC/WebSocket）会粘在某个后端。应做**按请求/按用户**分流。
- **两个版本并存时**：数据格式要双向兼容、会话与缓存不能串版本、监控要能按版本维度区分指标。
- **回滚只回退代码和流量，不回退数据**——上线前做好 schema 兼容设计才是根本。

### 33. 如何回滚？`kubectl rollout undo` 背后发生了什么？

**要点**：把 Deployment 的 Pod template 指回旧 ReplicaSet 的模板，再按滚动策略重新扩缩。

**展开**：
- `kubectl rollout undo deploy/x --to-revision=2` → 取历史上的 RS → 把其 `pod-template-hash` 对应的 template 抄回 Deployment.spec → Deployment controller 认为模板变更，触发一次新的 rollout（新 revision，但内容等于旧版）。
- 历史保留靠 `revisionHistoryLimit`（默认 10）；超过的旧 RS 被删除，就无法回滚到那个版本。
- `kubectl rollout restart` 只是加一个 `restartedAt` 注解，强制重建 Pod。
- 注意：**回滚不会回滚** ConfigMap/Secret 的引用变化（如果有同名 ConfigMap 已更新）、不会回滚数据库 schema、不会回滚 HPA 目标。

**面试加分点**：
- 「如何快速回滚？」保留旧镜像 tag + 幂等 schema；用 GitOps 回退 commit，比 `kubectl` 更可审计。
- 「回滚后还能再前进吗？」能，`rollout undo` 会生成新的 revision，历史是单向链但内容可重复。
- 「StatefulSet 能回滚吗？」不能自动 `rollout undo`；StatefulSet 的更新策略（`RollingUpdate`/`OnDelete`）和 partition 要手工管理，有状态回滚难度大得多。

### 34. liveness、readiness、startup 探针的区别？

**要点**：liveness 决定「重启」，readiness 决定「是否接流量」，startup 决定「启动期保护」。

**展开**：
- 探针类型：`httpGet`、`tcpSocket`、`exec`、`grpc`（1.24 GA）。
- 参数：`initialDelaySeconds`、`periodSeconds`、`timeoutSeconds`、`successThreshold`（liveness 必须为 1）、`failureThreshold`。
- liveness 失败 → kubelet 重启容器（`restartPolicy` 决定重启行为），可能连锁影响其他副本。
- readiness 失败 → Pod 从 EndpointSlice 移除，**不重启**；恢复后重新加入。
- startup 成功前会屏蔽 liveness/readiness，适合 Java/大模型加载等慢启动应用，避免 startup 期间被误杀。
- 最佳实践：
  - readiness 可以检查下游依赖（DB 连接、缓存），但要注意「依赖抖动 → 全量摘流 → 雪崩」。
  - liveness 只检查「进程是否卡死」，不要查外部依赖。
  - 用 `/healthz`（进程存活）+ `/readyz`（可接流量）分离。

**面试加分点**：
- 「探针为什么导致 Pod 反复重启？」`initialDelaySeconds` 太短、timeout 太短、探针检查了慢依赖。
- 「readiness 失败会导致滚动更新卡住吗？」会，新 Pod 不 Ready 时 Deployment 无法推进，表现为 `ProgressDeadlineExceeded`。
- 探针是由 kubelet 执行的，所以节点负载高时探针超时也会误判。

### 35. Pod 生命周期与常见的 Phase？

**要点**：调度 → 初始化 → 运行 → 终止；Phase 只是粗粒度状态，细节在 `Pod.Status.Conditions` 与容器状态里。

**展开**：
- Phase：`Pending`（已创建但未调度或镜像未就绪）、`Running`、`Succeeded`（所有容器成功退出且不再重启）、`Failed`、`Unknown`（节点失联）。
- 容器状态：`Waiting`（`ContainerCreating`、`ImagePullBackOff`、`CrashLoopBackOff`）、`Running`、`Terminated`（含 `exitCode`、`reason`：`OOMKilled`、`Error`、`Completed`）。
- 生命周期钩子：`postStart`（容器创建后，与主进程并发，失败会杀容器）、`preStop`（终止前，同步执行，超时后 SIGTERM）。
- 终止流程：`kubectl delete` → `deletionTimestamp` → 从 EndpointSlice 摘除（异步）→ 执行 preStop → SIGTERM → 等待 `terminationGracePeriodSeconds`（默认 30s）→ SIGKILL。
- Init 容器：按顺序执行，前一个成功才开始下一个（1.28+ 支持 `restartPolicy: Always` 的 sidecar 型 init 容器）。

**面试加分点**：
- 「Pod 处于 Running 但没 Ready？」看 readiness 探针与容器日志；Ready 是 Condition，不等于 Phase。
- 「Terminating 卡很久？」preStop 卡住、grace period 太长、finalizer、节点不可达。
- `pod.Spec.terminationGracePeriodSeconds` 与 `Deployment.spec` 不同，改 deployment 不影响已存在的 Pod。

### 36. 如何保证应用优雅退出？

**要点**：先停止接收新流量，再处理完在途请求，最后退出。

**展开**：
- 四步：
  1. 应用监听 `SIGTERM`，关闭监听端口（不再接新连接）。
  2. 等待在途请求完成（HTTP server 的 `graceful shutdown`）。
  3. 关闭连接池、释放资源、flush 日志。
  4. 退出（exit 0）。
- 配合 K8s：
  - `preStop` 里 `sleep 5`（等 EndpointSlice 摘除动作传播到所有 kube-proxy/Ingress）。
  - `terminationGracePeriodSeconds` 大于应用最大请求时长 + preStop 时长。
  - readiness 探针可以先失败（摘流），再退出。
- 长连接：WebSocket/gRPC 需要服务端主动 `GOAWAY` 或客户端重连策略；只靠 K8s 无法优雅。

**面试加分点**：
- 「preStop sleep 5 为什么能减少 5xx？」摘除 Endpoints 与 kube-proxy 规则更新是异步的，sleep 给传播留时间。
- 「为什么 SIGTERM 后还收到请求？」因为 EndpointSlice 更新有延迟，其他节点的 iptables 还没更新。
- 服务端不做 graceful shutdown 时，SIGTERM 直接退出会导致连接被 RST。

### 37. Init 容器与 Sidecar 的区别？

**要点**：Init 容器顺序执行且必须结束；Sidecar 与主容器并行运行，共享生命周期。

**展开**：
- Init 容器：`initContainers` 字段，按顺序串行，每个必须成功退出；失败时按 `restartPolicy` 重试；常用于等待依赖（DB 就绪）、下载配置、设置目录权限。资源 requests 会按 max(init, sum(app)) 计算。
- Sidecar（传统）：作为普通容器写在 `containers` 里，与主容器并行；缺点是无法保证启动顺序（常被 init 容器或 `postStart` 绕开）、无法在 Job 中正确处理。
- **原生 Sidecar**（1.29 beta，1.33 默认启用）：把 sidecar 写成 `initContainers` 且 `restartPolicy: Always`，保证它在主容器之前启动、在主容器之后退出，适用于 Istio/Linkerd 代理、日志 Agent。Job 场景下 sidecar 会随 Job 完成而退出，不会让 Job 卡住。

**面试加分点**：
- 「Sidecar 注入的时机？」Mutating Admission Webhook 在 Pod 创建时注入，或由 operator 直接改 Workload 模板。
- 「Sidecar 的 CPU/内存怎么设？」要考虑代理本身开销（Istio 大约 50–100MB 内存）；大流量场景需单独调参。
- 「为什么 Job 里用传统 Sidecar 会一直 Running？」主容器结束了，Sidecar 还在跑，Pod 不满足完成条件；原生 Sidecar 解决了这个问题。

### 38. 资源 requests 和 limits 的作用？

**要点**：requests 决定调度与保障，limits 决定运行上限；两者语义不同，不能混用。

**展开**：
- **requests**：调度器按它分配节点容量（`Allocatable`），也用于 QoS 分类；节点上 cgroup 的 `cpu.shares`/`memory.low` 以此为基础。
- **limits**：
  - CPU 超过 limit → CFS 节流（`throttled`），延迟升高但不会被杀。
  - 内存超过 limit → **OOMKilled**（cgroup OOM），退出码 137。
  - 不设 limit → 容器可用节点剩余资源，可能把节点打崩。
- 单位：CPU `m`（毫核），内存 `Ki/Mi/Gi`（二进制）或 `K/M/G`（十进制）。
- 扩展资源：`nvidia.com/gpu` 只能写整数，且 requests 与 limits 必须相等。

**面试加分点**：
- 「requests 设低、limits 设高有什么风险？」调度超卖，节点内存压力大时 Pod 被驱逐；CPU 争抢导致延迟抖动。
- 「limits 设成 requests 的一半会怎样？」稍微超卖，适合突发型应用；Guaranteed 场景应设相等。
- 「为什么 Java 应用 OOMKilled 但 JVM 堆没满？」容器内存包含 JVM 非堆（metaspace、线程栈、直接内存、GC 结构），JVM 需要设置 `-XX:MaxRAMPercentage` 感知容器限制（`UseContainerSupport`，JDK 10+ 默认）。

### 39. QoS 类别有哪些？

**要点**：Guaranteed > Burstable > BestEffort，节点压力下按优先级驱逐。

**展开**：
- **Guaranteed**：所有容器都设了 CPU 与内存的 requests == limits（且不为 0）。最稳，适合核心服务。
- **Burstable**：至少设置了部分 requests/limits，但不满足 Guaranteed。最常见。
- **BestEffort**：所有容器都没设任何 requests/limits。最先被驱逐。
- 驱逐顺序：`BestEffort` → `Burstable`（按使用量超出 requests 的比例）→ `Guaranteed`（最后）。驱逐由 kubelet 的 `eviction-hard`/`eviction-soft` 触发，对应 Node condition（`MemoryPressure`、`DiskPressure`、`PIDPressure`）。
- QoS 也影响 `oom_score_adj`：BestEffort 最高（最先被内核 OOM killer 选中）。

**面试加分点**：
- 「BestEffort 的 Pod 会被立刻杀吗？」不是，只在节点压力时；但内核 OOM 也可能先选中它。
- 「Guaranteed 就不会被驱逐吗？」不一定，`eviction` 阈值下如果节点整体内存耗尽，Guaranteed 也可能被杀，只是最后。
- 「如何排查被驱逐？」`kubectl describe pod` 的 `Status.Reason: Evicted`、节点 `kubectl describe node` 的 `Events`。

### 40. HorizontalPodAutoscaler 的工作原理？

**要点**：控制器周期性读取指标，按公式计算期望副本，再 `scale` 目标对象；受 `behavior` 策略约束。

**展开**：
- 公式：`desiredReplicas = ceil(currentReplicas × currentMetric / targetMetric)`（容忍度默认 10%，在 ±10% 内不扩缩）。
- 指标来源：`metrics-server`（`resource` 指标，如 CPU/内存）、`custom.metrics.k8s.io`（Prometheus Adapter）、`external.metrics.k8s.io`（KEDA/云监控）。
- 版本：`autoscaling/v2`（支持多指标、行为策略、内存指标）；`v1` 只支持 CPU。
- `behavior`：`scaleUp`/`scaleDown` 的 `stabilizationWindowSeconds`（防抖动）、`policies`（每 15s 最多扩 N 个）、`selectPolicy`。
- 与 VPA 的冲突：HPA 看工作负载实际用量，VPA 改 requests，二者同时用 CPU 指标容易抖（建议 HPA 用自定义 QPS 指标，VPA 只调内存）。
- 节点不够时的链路：HPA 扩 Pod → 节点资源不足 Pending → Cluster Autoscaler/Karpenter 扩容节点 → 调度成功。

**面试加分点**：
- 「HPA 为什么没扩容？」没设 requests（CPU 利用率算不出来）、metrics-server 不可用、Pod 不在 Running、目标值已达标、`behavior` 限制、`minReplicas` 已到上限。
- 「扩容后请求还是被拒？」扩出来的 Pod 需要时间（拉镜像、启动），需要提前预热或 `minReplicas` 保底。
- 「HPA 按内存扩缩可行吗？」可以但不推荐，内存不释放导致缩容不回去；用队列长度/QPS 更稳。

### 41. HPA 不生效的常见原因？

**要点**：先确认「指标有没有、能不能算、策略允不允许、目标能不能扩」。

**展开**：
1. **指标缺失**：`kubectl top pod` 失败 → metrics-server 未装/证书问题/网络不通；`kubectl get apiservice v1beta1.metrics.k8s.io` 看可用性。
2. **无法计算利用率**：Pod 没设 `requests.cpu`，HPA 报 `missing request for cpu`。
3. **目标对象状态**：Deployment 被 scale 到 0、HPA 与另一个控制器争抢 replicas、`scaleTargetRef` 名字写错。
4. **行为限制**：`behavior.scaleDown.stabilizationWindowSeconds` 太长，或 `policies` 卡住。
5. **指标延迟**：Prometheus Adapter 的 `minReplicas`/缓存、HPA 默认 15s 同步周期。
6. **已达到上下限**：`minReplicas`/`maxReplicas`。
7. **RBAC**：HPA 的 ServiceAccount 没有读 metrics/scale 子资源的权限。

**面试加分点**：
- 排查命令：`kubectl describe hpa x` 看 `Events`（`FailedGetResourceMetric`、`FailedComputeMetricsReplicas`）。
- 「HPA 与 PDB 无关」，别混淆；PDB 只限制自愿中断。
- 自定义指标需要 `custom.metrics.k8s.io` 的 APIService 注册，且指标必须有对应 label 匹配 Pod。

### 42. PodDisruptionBudget 是什么？

**要点**：限制「自愿中断」期间同时不可用的 Pod 数量，保证维护/升级时服务容量。

**展开**：
- 表达方式：`minAvailable`（最少可用，可以是数字或百分比）或 `maxUnavailable`（最多不可用），二者只能选一个。
- 生效范围：`selector` 匹配的 Pod；只对 **voluntary disruption**（drain、节点池升级、`evict` API）生效，不影响节点崩溃、OOM、滚动更新（Deployment 自己控节奏）。
- `drain` 时如果会违反 PDB，drain 会卡住并重试；`--disable-eviction` 会绕过 PDB（危险）。
- 与 HPA：PDB 基于副本数计算，扩缩容期间会动态变化。

**面试加分点**：
- 「PDB 配了 minAvailable: 100% 会怎样？」节点永远无法 drain，升级卡死；需要有冗余副本。
- 「单副本服务配 PDB」没有意义（任何中断都不可接受），应该多副本 + 反亲和。
- 「PDB 能防止滚动更新中断吗？」不能，Deployment 的 `maxUnavailable` 才负责这个；PDB 只保护 eviction。

### 43. 如何在 Pod 间共享配置与敏感信息？

**要点**：配置用 ConfigMap/Secret，通过 env 或 volume 注入；敏感数据优先用外部密钥系统。

**展开**：
- ConfigMap/Secret 注入方式：
  - `env` / `envFrom`（启动时快照，不热更新）。
  - volume 挂载（文件形式，kubelet 定期同步，可热更新，但应用需重读或配合 `subPath` 不推荐）。
  - `--from-file`/`--from-literal` 创建，`kustomize` 的 `configMapGenerator` 会带内容 hash，变更自动触发 rollout。
- 多容器共享：`emptyDir`（临时）、共享 ConfigMap 卷、`shareProcessNamespace`（共享 PID 命名空间，慎用）。
- 外部密钥：External Secrets Operator（同步 Vault/AWS Secrets Manager 到 K8s Secret）、Secrets Store CSI Driver（卷挂载，不落 etcd）、Vault Agent Injector（sidecar 注入）。

**面试加分点**：
- 「ConfigMap 更新后 Pod 会自动重启吗？」不会，除非用 hash 注解或 rollout restart；`subPath` 挂载也不会热更新。
- 「Secret 挂载为文件时是原子更新吗？」kubelet 用 symlink 切换目录，应用需要重新打开文件才能看到新内容（长驻进程需 watch）。
- 「配置错误导致 CrashLoop 怎么排查？」`kubectl exec` 进 Pod `cat /etc/config/...`，或直接看 ConfigMap 内容与挂载路径是否一致。

### 44. Downward API 能做什么？

**要点**：把「Pod 自身的信息」暴露给容器，而不需要调用 API Server。

**展开**：
- 可暴露字段：
  - 元数据：`metadata.name`、`namespace`、`uid`、`labels`、`annotations`。
  - 状态：`status.podIP`、`status.hostIP`、`status.podIPs`。
  - 容器资源：`limits.cpu`、`limits.memory`、`requests.*`。
  - `spec.nodeName`、`spec.serviceAccountName`。
- 两种模式：
  - 环境变量（`fieldRef`/`resourceFieldRef`），创建时定型。
  - volume 文件（`downwardAPI`），可部分热更新（labels/annotations）。
- 典型用途：日志中打上 Pod 名/节点名、JVM 按 `limits.memory` 设置堆大小（现在更推荐 `UseContainerSupport`）、应用启动时感知 AZ/节点。

**面试加分点**：
- 「想拿到 Service 的 ClusterIP 怎么办？」Downward API 拿不到 Service 信息，只能通过环境变量（K8s 会注入 `*_SERVICE_HOST`）或环境配置。
- 「Downward API 和 ConfigMap 注入的区别？」前者是系统信息，后者是用户配置。
- 大字段（完整 Pod spec）不适合放 Downward API，会增加 etcd 与 kubelet 压力。

### 45. Pod 的 `restartPolicy` 有几种？

**要点**：Always（默认）、OnFailure、Never；作用于 Pod 内**容器**，Pod 本身是否重建由上层控制器决定。

**展开**：
- `Always`：容器退出（无论成功失败）都重启，适合长期服务。
- `OnFailure`：非 0 退出才重启，适合 Job 的非幂等场景。
- `Never`：从不重启；Job 中若失败则整个 Job 失败（受 `backoffLimit` 影响）。
- 重启行为是**指数退避**：10s、20s、40s…最大 5 分钟，所以会看到 `CrashLoopBackOff` 状态。
- 重启计数与容器状态在 `Pod.Status.ContainerStatuses[].restartCount`。
- Deployment/StatefulSet/DaemonSet 创建 Pod 时 `restartPolicy` 必须是 `Always`；Job/CronJob 只能是 `OnFailure` 或 `Never`。

**面试加分点**：
- 「重启和重建的区别？」重启是同一 Pod 内重启容器，IP 不变；重建是删掉 Pod 换新 Pod（由控制器触发），IP 会变。
- 「CrashLoopBackOff 是状态还是原因？」是 kubelet 的等待状态，真正原因要看 `--previous` 日志和退出码。
- 「Pod 重启次数太多会怎样？」触发 `CrashLoopBackOff` 退避，最长 5 分钟一次；同时事件会刷屏，影响可观测性。

---

## 四、网络（46–60）

### 46. Kubernetes 网络模型的三条规则是什么？

**要点**：① Pod 之间无需 NAT 互通；② 节点与 Pod 无需 NAT 互通；③ Pod 看到的自 IP 与他人看到的一致。

**展开**：
- 这三条是 CNI 的「可移植性契约」：应用不需要知道自己跑在容器里，也不依赖端口映射。
- 「无需 NAT」指路由可达，不代表没有封装（VXLAN/IPIP 仍是同一地址空间，只是封包）。
- 允许的实现方式：路由（BGP）、Overlay（VXLAN）、云原生（VPC 路由/ENI）、eBPF 转发。
- 服务发现：ClusterIP 和 DNS 是额外抽象层，不属于三条规则。

**面试加分点**：
- 「为什么 Pod 不能用 hostNetwork 的端口冲突？」hostNetwork 共享节点网络命名空间，端口与节点冲突；且 Pod IP = 节点 IP。
- 「Pod 能 ping 通 Service VIP 吗？」通常不能，因为 iptables/IPVS 只处理 TCP/UDP，ICMP 不转发；IPVS 下某些实现会回 ICMP unreachable。
- 「跨节点为什么能通？」靠 CNI 配置的路由或隧道；若节点缺失到 Pod CIDR 的路由，就会表现为跨节点不通。

### 47. Pod 内多个容器如何通信？

**要点**：共享网络命名空间，通过 `localhost:port` 通信；不能监听同一端口。

**展开**：
- 共享内容：network namespace、IPC namespace（可选 `shareProcessNamespace` 共享 PID）、volume。
- 通信方式：`localhost` + 容器端口；共享文件用 `emptyDir`；共享内存用 `emptyDir.medium: Memory` + POSIX shm（注意 `sizeLimit`）。
- 端口冲突：两个容器不能都监听 8080（同一 ns）；可以给 sidecar 用 15001 这类专用端口。
- 进程信号：传统 Pod 内 `localhost` 是不同进程，需通过 HTTP 或共享文件通信；共享 PID 后可 `kill` 对方进程。

**面试加分点**：
- 「Pod 内 DNS 解析谁提供？」Pod 的 `/etc/resolv.conf` 由 kubelet 按 `dnsPolicy` 生成，指向 kube-dns Service 的 ClusterIP。
- 「服务网格为什么能劫持流量？」iptables 规则把出站流量重定向到 sidecar 的 15001，入站到 15006。
- 「多容器 Pod 的日志怎么区分？」`kubectl logs pod -c container`，日志采集器按容器分别采集。

### 48. 跨节点 Pod 通信是怎么实现的？

**要点**：由 CNI 决定：Overlay 封装、Underlay 路由（BGP）、或云厂商 ENI/VPC 路由。

**展开**：
- **Overlay（VXLAN/IPIP）**：Pod 包被封装进节点 IP 的 UDP 包（VXLAN 4789），在节点间传输，MTU 需要减 50 字节；简单、与底层网络无关，但有封装开销。
- **Underlay/BGP（Calico）**：节点间通过 BGP 宣告 Pod CIDR，数据包不封装，性能好；要求底层网络支持或与路由器对接。
- **云原生（AWS VPC CNI / Azure CNI）**：Pod 直接从 VPC 子网拿 IP，走原生 VPC 路由；无封装性能好，但消耗 IP 且受子网容量限制。
- **IPIP**：Calico 的一种折中，开销较小（20 字节）。
- **kube-router / Antrea**：也支持 BGP/OVS 方案。

**面试加分点**：
- 「Pod CIDR 怎么分？」kube-controller-manager 的 `--cluster-cidr` + 每个节点 `--node-cidr-mask-size`，节点从 Node 对象的 `spec.podCIDR` 拿到自己的网段。
- 「跨节点不通但同节点通？」多半是路由/BGP 断开或安全组/防火墙阻断封装端口（VXLAN 4789/UDP、IPIP）。
- 「MTU 设置在哪？」CNI 配置（如 Calico 的 `veth_mtu`）、节点网卡 MTU；不一致会导致大包丢失。

### 49. Service 的 DNS 记录长什么样？

**要点**：`<service>.<namespace>.svc.<cluster-domain>`，默认 `cluster.local`。

**展开**：
- A/AAAA：ClusterIP 类型返回 VIP；Headless 返回所有就绪 Pod IP。
- SRV：`_<port-name>._<protocol>.<service>.<ns>.svc.cluster.local`，用于发现端口。
- Pod 记录：`<ip>.<ns>.pod.cluster.local`（默认关闭，kubelet `--enable-pod-dns` 或 CoreDNS 配置）。
- StatefulSet：`<pod>.<service>.<ns>.svc.cluster.local`，需要 Headless Service。
- 搜索域：Pod 的 `/etc/resolv.conf` 中 `search <ns>.svc.cluster.local svc.cluster.local cluster.local`，所以同 namespace 下只写 `svc` 即可。
- 短名解析：`my-svc`、`my-svc.my-ns`、`my-svc.my-ns.svc` 均可。

**面试加分点**：
- 「ExternalName 的 DNS 长什么样？」返回 CNAME，客户端解析到外部域名。
- 「为什么 Pod 内 ping `my-svc` 不通？」ClusterIP 由 iptables 处理，ICMP 不转发；用 `curl`/`nc` 测。
- `ndots:5` 导致每个短域名先尝试多个 search 后缀，长域名解析慢且产生大量 NXDOMAIN。

### 50. CoreDNS 的作用与常见问题？

**要点**：集群默认 DNS 服务，把 Service/Pod 名解析成 IP；故障会导致大面积「解析失败」。

**展开**：
- 部署形态：`kube-system` 下的 Deployment（通常 2+ 副本）+ `kube-dns` Service（ClusterIP）+ ConfigMap `coredns`（Corefile）。
- 插件链：`errors`、`health`、`ready`、`kubernetes`（处理集群内域名）、`prometheus`、`forward . /etc/resolv.conf`（转发外部域名）、`cache`、`loop`。
- 常见问题：
  - 副本不足/调度到单节点 → 单点故障。
  - NetworkPolicy 阻断到 kube-dns 的 UDP/TCP 53。
  - `ndots:5` 放大查询量。
  - 上游 DNS 慢或不可达。
  - conntrack 表满导致 UDP 丢包（表现为「间歇性 DNS 超时」）。
  - NodeLocal DNSCache 未部署，所有 Pod 直连 CoreDNS Pod。

**面试加分点**：
- 「如何缓解 DNS 压力？」部署 NodeLocal DNSCache（每节点一个缓存 DaemonSet，Pod 指向 169.254.20.10）、调小 `ndots`、开启 CoreDNS `cache`、用 `autopath`/FQDN。
- 「CoreDNS 的 request 来源？」`kubectl top pod -n kube-system -l k8s-app=kube-dns` + `coredns_dns_request_count_total` 指标。
- 「解析外部域名超时？」检查 `forward` 的上游、节点 `/etc/resolv.conf`、是否有 NetworkPolicy 阻断 egress。

### 51. `ndots:5` 和 `search` 域导致的问题如何优化？

**要点**：短域名会先尝试多个 search 后缀，产生大量 NXDOMAIN；用 FQDN 或降低 `ndots` 优化。

**展开**：
- `ndots:5` 含义：如果域名中的点少于 5 个，先尝试 search 列表（`<ns>.svc.cluster.local`、`svc.cluster.local`、`cluster.local`、上游域名）再尝试绝对域名。
- 访问 `api.example.com`（2 个点）时，会依次查询：
  `api.example.com.<ns>.svc.cluster.local` → `api.example.com.svc.cluster.local` → `api.example.com.cluster.local` → `api.example.com`，共 4 次。
- 优化：
  - 用 FQDN（`api.example.com.`，末尾加点）跳过 search。
  - Pod `dnsConfig.options[].name: ndots, value: "2"`。
  - NodeLocal DNSCache + 缓存。
  - 对高频外部依赖使用 IP 或长连接。
- `ndots` 由 kubelet 的 `--cluster-dns` 与 `dnsConfig` 共同决定，默认 5（源于 K8s 组件自身需要）。

**面试加分点**：
- 「为什么 ndots 默认 5？」为了兼容 `service.namespace.svc.cluster.local` 这类长域名，让集群内名字优先。
- 「降低 ndots 有什么风险？」集群内名字解析可能多一次 `cluster.local` 的绝对查询，但通常影响可忽略。
- 「怎么量化影响？」CoreDNS 指标 `coredns_dns_responses_total{rcode="NXDOMAIN"}` 很高就是典型症状。

### 52. NetworkPolicy 的作用与限制？

**要点**：基于 label 的四层准入控制，必须由 CNI 实现；只做 L3/L4，不做七层。

**展开**：
- 字段：`podSelector`（作用对象）、`policyTypes`（Ingress/Egress）、`ingress.from`（podSelector/namespaceSelector/ipBlock）、`egress.to`（同上 + `ports`）。
- 默认拒绝：创建 `podSelector: {}` 且 `policyTypes: [Ingress, Egress]` 的空策略，选中所有 Pod，拒绝所有入/出（除 DNS 需要显式放行）。
- 语义是「白名单」：一旦某个 Pod 被选中，未被任何规则允许的方向流量全部拒绝。
- 实现依赖：Calico、Cilium、Antrea、Kube-router 支持；**Flannel 原生不支持**（需要 Calico 一起用）。
- 不支持的场景：七层（HTTP 方法/路径）、域名（部分实现支持 FQDN）、HostNetwork Pod（绕过策略）、非 Pod 流量（默认只作用于 Pod 网络）。

**面试加分点**：
- 「NetworkPolicy 为什么不生效？」CNI 不支持、Pod 在 hostNetwork、策略写错 namespaceSelector（空表示所有 ns）、选择器没匹配到。
- 「如何做七层策略？」Cilium `CiliumNetworkPolicy`（L7 HTTP/gRPC）、Istio `AuthorizationPolicy`。
- 「集群默认拒绝怎么做？」`kube-system` 也要小心放行（DNS、metrics），不要一刀切。

### 53. NetworkPolicy 默认规则是什么？

**要点**：没有策略选中 → 全放行；一旦被选中 → 只放行显式允许的流量。

**展开**：
- 三种组合：
  1. 没有任何 NetworkPolicy：所有 Pod 互通（默认）。
  2. 存在 `podSelector: {}` + `policyTypes: [Ingress]`：所有 Pod 的入站被拒，除非规则允许（甚至同 namespace）。
  3. 只选中部分 Pod：只有这些 Pod 受限，其他不受影响。
- Ingress 与 Egress 独立：只写 Ingress 策略时，出站不受限。
- 「允许同 namespace 通信」的写法：
  ```yaml
  ingress:
    - from:
        - podSelector: {}
  ```
- DNS 放行（Egress 策略必须显式加）：
  ```yaml
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
  ```

**面试加分点**：
- 「写了 NetworkPolicy 但流量还是通？」检查 CNI 是否支持、是否被 `hostNetwork` 绕过、策略有没有被 `policyTypes` 限定方向。
- 「namespaceSelector 为空和 podSelector 为空的区别？」`namespaceSelector: {}` 表示所有 namespace；`podSelector: {}` 表示本策略所在 namespace 的所有 Pod。
- 默认拒绝是生产基线，但需要配合白名单逐步迁移（先用 audit 模式或 Cilium 的 policy audit）。

### 54. 如何排查「Pod 之间连不通」？

**要点**：从 DNS → 策略 → 监听 → Service/Endpoints → 路由 → MTU，逐层收敛。

**展开**：
- 排查清单：
  1. **DNS**：`nslookup svc`、`getent hosts`、`cat /etc/resolv.conf`。
  2. **NetworkPolicy**：`kubectl get netpol -A`，对照 selector 与目标端口。
  3. **目标监听**：`kubectl exec -it pod -- ss -lntp`（或 `netstat`）。
  4. **Service 与 Endpoints**：`kubectl get endpointslices`，看是否有后端、端口是否正确。
  5. **直连 Pod IP**：绕过 Service，`curl <podIP>:port`，区分「Service 问题」还是「Pod 网络问题」。
  6. **CNI 与路由**：`ip route`、`iptables -t nat -L`、`cilium status`/`calicoctl node status`。
  7. **跨节点**：`tcpdump -i any port <port>` 在源/目标节点抓包，看 SYN 是否到达。
  8. **MTU**：`ping -M do -s 1400 <ip>` 测大包。
- 常用工具：`kubectl debug`（ephemeral container）、`netshoot` 镜像、`cilium hubble`。

**面试加分点**：
- 「同节点通、跨节点不通」→ CNI 路由/隧道/防火墙。
- 「小包通、大包不通」→ MTU。
- 「Service 通、Pod IP 不通」→ Service/Endpoints/iptables/IPVS 问题。
- 「DNS 通但 TCP 不通」→ NetworkPolicy 或 conntrack。

### 55. 什么是 MTU 问题？

**要点**：封装会增加包头，若 Pod 网卡 MTU 未相应调小，大包会被丢弃且没有明确报错。

**展开**：
- 常见开销：VXLAN 50 字节（8 VXLAN + 8 UDP + 20 IP + 14 Eth），IPIP 20 字节，WireGuard 约 60 字节。
- 默认 1500 的节点上，VXLAN 后 Pod MTU 应为 1450；Calico 默认会计算（`veth_mtu` 或 `ipv4` 的 `mtu`）。
- 症状：小请求（DNS、TCP 握手）正常，大响应（TLS 证书、大 JSON、文件传输）卡住或超时；`ping -s 1472` 可复现。
- 排查：`ip link` 看 Pod 网卡 MTU、节点网卡 MTU、`ping -M do -s <payload>`、`tracepath`。
- 修复：统一 CNI 与节点 MTU、开启 TCP MSS clamping（iptables `TCPMSS --clamp-mss-to-pmtu`，kube-proxy 默认有）。

**面试加分点**：
- 「为什么容器里 MTU 是 1450 而节点是 1500？」CNI 按封装开销计算；跨集群/混合网络时更容易出问题。
- 「K8s 默认有 MSS clamping 吗？」kube-proxy 的 `--tcp-mss-clamp` 可为 ClusterIP 流量设置；hostNetwork/NodePort 场景需额外规则。
- 「怎么用最小代价确认 MTU？」`kubectl exec` 里 `ping -M do -s 1400 -c 1 <对端>`，逐步加大。

### 56. Service 没有 Endpoints 的常见原因？

**要点**：Selector 不匹配、Pod 不 Ready、namespace 不对、端口名错、Pod 正在终止。

**展开**：
- 逐项排查：
  1. `kubectl get svc -o yaml` 看 `spec.selector`。
  2. `kubectl get pods -l <selector> -n <ns> -o wide` 是否匹配。
  3. `kubectl get pod <pod> -o yaml` 看 `status.conditions` 中 `Ready` 是否为 True。
  4. Pod 是否 `Terminating`（默认只有 Ready 且非 Terminating 才进 EndpointSlice）。
  5. Service 与 Pod 是否同一 namespace（跨 ns 必须用 ExternalName，不能直接选）。
  6. 是否使用了 `publishNotReadyAddresses: true`（StatefulSet 集群发现需要）。
  7. EndpointSlice controller 是否正常、是否有 `kubernetes.io/service-name` label。
  8. 是否用了 `endpointSlice` 手动管理（无 selector 的 Service + 手写 EndpointSlice）。
- 命名端口：`service.port.targetPort` 要对应容器端口名或数字。

**面试加分点**：
- 「Pod 在 Running 但不 Ready」是第一名原因，多数是 readiness 探针失败。
- 「Endpoints 有但访问不通」多为端口不匹配或 NetworkPolicy。
- 「无 selector 的 Service」用于对接外部服务或自管 EndpointSlice（如 `ExternalName` 替代方案）。

### 57. ExternalTrafficPolicy 的 Local 与 Cluster 区别？

**要点**：Cluster 会二次转发到任意节点 Pod（丢源 IP）；Local 只转发到本节点 Pod（保留源 IP，但可能不均衡）。

**展开**：
- **Cluster**（默认）：外部流量到节点 A，A 上的 kube-proxy 可能 DNAT 到节点 B 上的 Pod，多一跳，**源 IP 被 SNAT 成节点 IP**（除非开启 `preserveClientIP` 相关的 LB 特性）。
- **Local**：只把流量转到本节点上的 Pod；若无本节点 Pod，流量被丢弃（表现为部分节点访问失败）。用于：
  - 需要客户端源 IP（审计、限流、GeoIP）。
  - 减少跨节点一跳（延迟敏感）。
- 配合云 LB 的健康检查：`Local` 时 LB 只把流量给有 Pod 的节点（`healthCheckNodePort`）。
- `internalTrafficPolicy: Local` 对集群内部流量同理（1.22+）。

**面试加分点**：
- 「Local 模式下副本少时会有节点黑洞」——需要足够的副本 + 反亲和或用 `topologySpread`，或者用 LB 的节点过滤。
- 「源 IP 丢失怎么恢复？」LB 层开启 proxy protocol、Ingress 的 `use-forwarded-headers`、或 `Local` 模式。
- 「NodePort 用 Local 时，健康检查端口是 `healthCheckNodePort`」，这是 CCM 创建的。

### 58. sessionAffinity 如何实现？

**要点**：Service 的 `sessionAffinity: ClientIP`，kube-proxy 基于源 IP 做粘性转发，超时由 `sessionAffinityConfig.clientIP.timeoutSeconds` 控制。

**展开**：
- 实现依赖 conntrack：同一源 IP 的后续连接在超时内命中同一条 DNAT 规则。
- 粒度问题：NAT/代理后大量客户端共享源 IP，粘性失效或过度集中；IPv6/DSR 场景支持有限。
- 更好的替代：
  - 七层：Ingress/网关的 cookie 粘性（`nginx.ingress.kubernetes.io/affinity: cookie`）。
  - 网格：Istio `DestinationRule.trafficPolicy.loadBalancer.consistentHash`（基于 header/cookie）。
  - 应用层：分布式 session（Redis）而不是粘性。
- Headless Service 下 kube-proxy 不参与，客户端自己选后端，粘性由客户端实现。

**面试加分点**：
- 「ClientIP 粘性为什么在 kube-proxy 重启后丢失？」conntrack 表被清空。
- 「为什么推荐去掉粘性？」粘性会导致负载不均和扩容无效，session 共享才是根本解。
- gRPC 长连接本质上就是「天然粘性」，需要客户端 LB 或 `round_robin` resolver。

### 59. 七层负载均衡与四层负载均衡的区别？

**要点**：四层看 IP/端口，七层看 HTTP 语义；前者快，后者智能。

**展开**：
- 四层：TCP/UDP 转发（Service、云 NLB、LVS），不解析载荷，性能高、无状态、适合任意协议；无法按 URL 路由、无法做 HTTP 重试/超时策略。
- 七层：解析 HTTP/HTTPS（Ingress、Envoy、Nginx、ALB），支持 Host/Path 路由、TLS 终止与 SNI、Header 改写、重试、限流、熔断、灰度、WAF。
- 性能代价：七层要终止/解析连接，CPU 与内存开销大；长连接场景还要考虑连接复用与 `keepalive`。
- 组合使用：外部 L4（云 LB）→ 内部 L7（Ingress/网关）。
- 在 K8s 中：Service 是 L4（IPVS/iptables），Ingress/Gateway API 是 L7。

**面试加分点**：
- 「WebSocket 走七层还是四层？」七层网关需要支持 Upgrade；四层更简单。
- 「gRPC 需要 HTTP/2，Ingress 支持吗？」需要 Controller 支持 h2c/HTTP2，注意 `grpc` 注解与后端协议（`backend-protocol: GRPC`）。
- 「七层如何保留客户端 IP？」`X-Forwarded-For`/`X-Real-IP` + 应用信任链配置（`trusted proxies`）。

### 60. Gateway API 是什么？为什么提出它？

**要点**：Ingress 的演进，按角色拆分资源、支持更多协议与跨 namespace 路由。

**展开**：
- 资源模型：
  - `GatewayClass`：由基础设施提供方实现（相当于 IngressClass）。
  - `Gateway`：一个具体的入口（监听端口、TLS、地址）。
  - `HTTPRoute`/`TCPRoute`/`TLSRoute`/`GRPCRoute`：路由规则，可跨 namespace 引用 Gateway（`allowedRoutes`）。
- 相比 Ingress 的改进：
  - **角色分离**：基础设施团队管 Gateway，应用团队管 Route。
  - **表达力**：路径/header/query 匹配、权重、重写、重定向、镜像、超时。
  - **协议**：原生 TCP/UDP/TLS/gRPC。
  - **可移植性**：核心字段标准化，厂商扩展用 `Policy`（如 `BackendTLSPolicy`）。
- 状态：核心 API 已 GA（1.1+），`GRPCRoute`/`BackendTLSPolicy` 等逐步 GA；Nginx、Envoy Gateway、Istio、Cilium 等已实现。

**面试加分点**：
- 「Ingress 会被淘汰吗？」短期内共存，Gateway API 是新项目的推荐方向。
- 「Gateway 和 Service 的关系？」Route 引用 Service 作为后端，Gateway 提供 Listeners。
- 「如何做多租户？」每个团队一个 namespace + HTTPRoute + Gateway 的 `allowedRoutes.namespaces.from: Selector`。

---

## 五、存储（61–70）

### 61. PV、PVC、StorageClass 的关系？

**要点**：PVC 是「申请」，PV 是「资源」，StorageClass 是「怎么造资源」的模板；动态供给把三者串起来。

**展开**：
- PVC：`accessModes`、`resources.requests.storage`、`storageClassName`、`selector`。
- PV：`capacity`、`accessModes`、`persistentVolumeReclaimPolicy`（Retain/Delete/Recycle）、`storageClassName`、`claimRef`、`nodeAffinity`（拓扑）。
- StorageClass：`provisioner`（CSI 驱动名）、`parameters`（磁盘类型、IOPS）、`reclaimPolicy`、`volumeBindingMode`、`allowVolumeExpansion`、`mountOptions`。
- 绑定过程：静态（管理员预建 PV）或动态（provisioner 造 PV）。PVC 与 PV 必须容量 ≥ 请求、accessMode 兼容、storageClassName 相同。
- `default` StorageClass：给 SC 打 `storageclass.kubernetes.io/is-default-class: "true"`，PVC 不写 `storageClassName` 时使用。

**面试加分点**：
- 「PVC 一直 Pending？」没有可用 PV、SC 不存在、provisioner 失败、`WaitForFirstConsumer` 等调度、配额限制。
- 「Retain vs Delete？」Delete 删 PVC 时删后端卷（云盘会丢数据）；Retain 保留卷，需手动回收。
- 「一个 PV 能被多个 PVC 用吗？」不能，一对一绑定；`ReadWriteMany` 是多个 Pod 共用一个 PVC。

### 62. 访问模式有哪几种？

**要点**：RWO/ROX/RWX/RWOP，注意 RWO 是**节点级**而不是 Pod 级。

**展开**：
- `ReadWriteOnce`：单节点读写（云盘、本地盘最常见）。
- `ReadOnlyMany`：多节点只读（如 NFS 共享只读镜像）。
- `ReadWriteMany`：多节点读写（NFS、CephFS、EFS、GlusterFS）。
- `ReadWriteOncePod`（1.22+，1.27 GA）：单 Pod 读写，用于避免同节点多 Pod 争用同一卷（如 SQLite）。
- 约束：访问模式由底层存储能力决定，声明超过支持的会绑定失败。
- 拓扑：云盘通常绑定到某个 zone，Pod 调度需同 zone（`WaitForFirstConsumer` 或 `nodeAffinity`）。

**面试加分点**：
- 「RWO 卷能给同节点两个 Pod 挂吗？」技术上可以（node 级锁），但存在并发写风险，推荐 RWOP。
- 「RWX 性能」：NFS 类方案在高并发写时性能有限；共享文件系统要评估 inode 与带宽。
- 「RWO 的 Pod 漂移到别的节点？」云盘会重新 attach，但 attach 有延迟；本地盘（local PV）无法漂移。

### 63. 动态供给的流程？

**要点**：PVC 挂到 SC → external-provisioner 调 CSI 建卷 → 生成 PV 并绑定 → 调度 → attach/mount。

**展开**：
- 详细步骤：
  1. 用户创建 PVC（指定 SC）。
  2. **PV controller**（KCM 内）发现 PVC 无对应 PV，读取 SC，交给 external-provisioner。
  3. **external-provisioner**（CSI sidecar）调用 `CreateVolume`，云上创建磁盘。
  4. 生成 PV 对象，写入 PVC 的 `spec.volumeName`，PVC 变为 `Bound`。
  5. 创建 Pod → 调度器考虑卷拓扑（`WaitForFirstConsumer` 时先调度再建卷）。
  6. **external-attacher** 调 `ControllerPublishVolume`，把卷 attach 到节点（云盘常见）。
  7. kubelet 调 **node 插件** 的 `NodeStageVolume`（格式化/挂载到全局目录）和 `NodePublishVolume`（bind mount 到 Pod 目录）。
  8. Pod 删除时反向执行，最终 `DeleteVolume`（取决于 reclaimPolicy）。

**面试加分点**：
- 「Pod Pending 报 `volume node affinity conflict`」——卷在某 zone，Pod 被调度到别处。
- 「Pod 卡在 `FailedMount`」——CSI node 插件缺失、挂载点陈旧、SELinux、文件系统损坏。
- 「为什么 PVC 建好了但 Pod 还是 Pending？」`WaitForFirstConsumer` 场景下是正常的，等调度后才建卷。

### 64. 什么是 VolumeBindingMode `WaitForFirstConsumer`？

**要点**：延迟绑定，等 Pod 调度后再创建/绑定卷，避免卷与节点拓扑不匹配。

**展开**：
- 默认 `Immediate`：PVC 创建就 provisioning，卷在随机可用的 zone（如 `us-east-1a`），Pod 随后可能被调度到 `1b` → `volume node affinity conflict`。
- `WaitForFirstConsumer`：调度器在 Filter 阶段把「卷将在哪些 zone 可用」纳入考虑，选出节点后再建卷/绑定。适合多可用区、云盘、本地 PV（`local` 卷）。
- 也适用于 `local` PV 的 `nodeAffinity` 场景。
- 注意：PVC 在没 Pod 时一直 Pending 是正常的。

**面试加分点**：
- 「所有 SC 都该用 WaitForFirstConsumer 吗？」多 zone 场景必要；单 zone 或共享存储（NFS/EFS）无所谓。
- 「Immediate 的修复方式？」改 SC 或手动重建 PVC；已绑定的 PVC 不能改 binding mode。
- 「与 `topologySpreadConstraints` 的交互？」调度器会同时考虑卷拓扑与分布约束，可能出现无解 → Pending。

### 65. StatefulSet 的存储如何工作？

**要点**：`volumeClaimTemplates` 为每个 Pod 生成独立 PVC，Pod 重建后保留数据。

**展开**：
- 命名：`<volumeClaimTemplate.name>-<statefulset.name>-<ordinal>`，如 `data-mysql-0`。
- 生命周期：PVC 与 StatefulSet **解耦**，删除 StatefulSet 不会删 PVC（除非手动或加 finalizer/`persistentVolumeClaimRetentionPolicy`，1.27 beta）。
- Pod 重建/重调度：新 Pod 挂载同一个 PVC，数据保留；但 **Pod IP 会变**，靠 DNS 名（Headless Service）保持身份。
- 扩容：`replicas` 增加会按序创建新 Pod + 新 PVC；缩容不会自动删 PVC（避免误删数据）。
- 常见坑：zone 漂移（PVC 在某 zone，Pod 调度到别的 zone 导致 Pending）。

**面试加分点**：
- 「删 StatefulSet 后数据还在吗？」在，PVC 还在；恢复时重建 StatefulSet 会重新绑定已有 PVC（名字要一致）。
- 「StatefulSet 能滚动更新吗？」能，`RollingUpdate` 逆序更新；但数据库升级通常需要 Operator 协调主从。
- 「为什么 StatefulSet 要配 Headless Service？」提供稳定的 `pod-0.svc` DNS，且客户端可发现所有副本。

### 66. emptyDir 和 hostPath 的区别？

**要点**：emptyDir 随 Pod 生死，hostPath 绑定节点目录。

**展开**：
- `emptyDir`：Pod 创建时创建，Pod 删除时删除；可 `medium: Memory`（tmpfs，计内存）；`sizeLimit` 限制大小。用于容器间共享、缓存、临时文件。
- `hostPath`：挂载节点上的目录/文件；`type` 有 `DirectoryOrCreate`/`File`/`Socket` 等。Pod 删除数据仍在，但绑死节点。
- hostPath 的用途：DaemonSet 访问 `/var/log`、`/var/lib/docker`、`/etc/kubernetes`；本地设备。
- 风险：安全（容器可读写宿主机）、调度（Pod 换了节点数据就没了）、多 Pod 冲突。

**面试加分点**：
- 「hostPath 在生产怎么用？」只给 DaemonSet 的 node 级组件，配合 PSA `baseline` 限制。
- 「local PV 和 hostPath 的区别？」local PV 有 PV/PVC 抽象、有 nodeAffinity、生命周期由存储管理；hostPath 没有。
- 「emptyDir 的 InMemory 会 OOM 吗？」会，计入 Pod 内存限制。

### 67. 如何扩展 PVC 容量？

**要点**：SC 开 `allowVolumeExpansion: true`，patch PVC 的 `requests.storage`，驱动执行扩容。

**展开**：
- 步骤：
  1. `kubectl patch sc <sc> -p '{"allowVolumeExpansion": true}'`（或直接在 SC 上设置）。
  2. `kubectl patch pvc <pvc> -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'`。
  3. CSI `external-resizer` 调 `ControllerExpandVolume`，文件系统扩容由 `NodeExpandVolume` 完成。
- 限制：只能扩不能缩（K8s 不支持 shrink）；不是所有驱动支持在线扩容（部分需要重启 Pod 或卸载）。
- 版本：从 1.24 起 GA；`ExpandInUsePersistentVolumes` 允许多个 Pod 在使用时扩容。
- 观察：PVC 的 `status.conditions` 里有 `FileSystemResizePending`。

**面试加分点**：
- 「扩容一直 Pending？」SC 未开、驱动不支持、卷正在使用且不支持 online resize。
- 「缩容怎么办？」新建 PVC + 数据迁移（rsync/备份恢复）。
- 「扩容后容器里看不到新空间？」文件系统未 resize，检查 `df -h` 与 PVC condition。

### 68. 如何在 Kubernetes 中运行有状态数据库？

**要点**：StatefulSet + 稳定网络 + 独立 PVC + Operator 管理生命周期。

**展开**：
- 基础：StatefulSet（稳定序号/域名）、Headless Service（`pod-N.svc`）、PVC 模板、PDB、反亲和（跨 zone）、资源 Guaranteed。
- Operator 做的事：主从/副本选举、故障切换（promote/rejoin）、备份恢复、版本升级（滚动）、扩缩容与再平衡、监控指标。
- 常见选择：
  - PostgreSQL：CloudNativePG、Zalando Postgres Operator。
  - MySQL：Vitess（分片）、MySQL Operator。
  - Redis：Redis Operator、Sentinel/Kubernetes-native。
  - Kafka：Strimzi。
  - Elasticsearch：ECK。
- 外部数据面：很多团队用「K8s 跑业务 + 云托管 DB」，减少有状态运维复杂度。

**面试加分点**：
- 「数据库能跑在 K8s 里吗？」能，但要有 Operator、存储性能保障（本地 NVMe/local PV）、网络稳定性（避免频繁迁移）、备份演练；否则不如托管。
- 「Pod 漂移后数据怎么办？」PVC 跟 Pod 走（云盘跨节点 attach），但本地盘不行。
- 「如何做故障切换？」Operator 用 Lease/选举 + 应用层探针，避免脑裂。

### 69. 备份策略怎么设计？

**要点**：资源与 PV 备份、应用级备份、etcd 快照、异地存储、恢复演练。

**展开**：
- 层次：
  1. **集群资源**：Velero 备份 YAML（含 Secret、CRD、ClusterRole），可恢复到其他集群。
  2. **持久卷**：Velero 的 Restic/Kopia 或 CSI 快照（`VolumeSnapshot`）。
  3. **数据库**：逻辑备份（pg_dump、mysqldump）+ 物理备份（xtrabackup、WAL 归档），保证一致性。
  4. **etcd**：`etcdctl snapshot save`，灾备恢复控制面。
- 策略：RPO/RTO 定义、频率、保留期（GFS 3-2-1）、加密、异地/跨账号、访问控制。
- 恢复演练：定期在隔离环境 restore，验证可恢复性（备份不算数，能恢复才算）。
- 注意：备份要包含「集群级资源」（CRD、RBAC、StorageClass），否则恢复后应用起不来。

**面试加分点**：
- 「Velero 备份 Secret 吗？」默认备份，注意加密与 RBAC。
- 「跨集群恢复的坑？」UID 引用、ServiceAccount token、云资源名冲突。
- 「如何保证备份一致性？」一致性快照（CSI snapshot）或应用停止写入（`pre-backup` hook）。

### 70. 为什么 Pod 会一直 ContainerCreating / 挂载失败？

**要点**：绝大多数是镜像拉取、卷挂载、Sandbox 创建、网络配置卡住。

**展开**：
- 排查路径：
  1. `kubectl describe pod` 看 `Events`（`FailedMount`、`FailedAttachVolume`、`FailedCreatePodSandBox`、`ImagePullBackOff`）。
  2. 卷：PVC 是否 Bound、CSIS 驱动 Pod 是否正常、卷与节点是否同 zone、SELinux/fsGroup 权限、挂载点是否陈旧。
  3. Secret/ConfigMap：被引用的对象不存在且非 optional 时，Pod 会一直等待。
  4. 运行时：containerd 拉镜像慢、磁盘满（`no space left on device`）。
  5. CNI：IP 耗尽、插件不可用。
- 常用命令：`kubectl get events --sort-by=.lastTimestamp`、`journalctl -u kubelet`、`crictl ps -a`。

**面试加分点**：
- 「为什么挂载会一直重试？」kubelet 的 volume manager 有退避机制，最长 2 分钟一次；事件里能看到退避信息。
- 「`Terminating` 的 Pod 还占着卷？」会阻塞新 Pod 的挂载（RWO），需要先处理旧 Pod。
- 「`FileSystemResizePending` 也会导致创建卡住」，尤其在扩容后的 PVC 上。

---

## 六、安全（71–82）

### 71. 认证与授权在 Kubernetes 中如何工作？

**要点**：认证「你是谁」→ 授权「能做什么」→ 准入「还允许什么」。

**展开**：
- 认证：
  - 客户端证书（`--client-ca-file`，CN 是 user，O 是 group）。
  - Bearer Token（静态 token 文件已不推荐、ServiceAccount JWT、OIDC ID Token）。
  - Webhook 认证（对接外部 IAM）。
  - 匿名认证（默认对部分路径开放，可 `--anonymous-auth=false` 关闭）。
- 授权：
  - **RBAC**（主流）：Role/ClusterRole + Binding。
  - **Node**：kubelet 只能操作自己的资源。
  - **Webhook**：对接 OPA/cloud IAM。
  - **ABAC**：文件策略，已不推荐。
- 准入：Mutating/Validating Admission Webhook、PSA、ResourceQuota、LimitRange、NodeRestriction。
- 顺序：认证 → 授权 → 准入（Mutating → 校验 → Validating）→ 存储。

**面试加分点**：
- 「如何验证权限？」`kubectl auth can-i get pods --as=system:serviceaccount:ns:sa`、`kubectl auth can-i --list`。
- 「OIDC 与 SA 的区别？」OIDC 是「人」，SA 是「Pod/工作负载」。
- 「kubelet 的 Node 授权器」防止 kubelet 冒充其他节点。

### 72. RBAC 的核心对象？

**要点**：Role/ClusterRole 定义「动词 + 资源」，Binding 把它绑定到「主体」。

**展开**：
- 权限定义：`apiGroups`、`resources`（`pods`、`pods/log`、`deployments`、`*`）、`verbs`（get/list/watch/create/update/patch/delete/deletecollection）、`resourceNames`（细到对象名）、`nonResourceURLs`（如 `/healthz`）。
- 绑定：
  - RoleBinding（namespace 级）→ 绑定 Role 或 ClusterRole（在 namespace 内生效）。
  - ClusterRoleBinding（全局）→ 只能绑 ClusterRole。
- 主体：`User`、`Group`、`ServiceAccount`（`system:serviceaccount:<ns>:<name>`）。
- 常用 ClusterRole：`view`、`edit`、`admin`、`cluster-admin`（避免给日常工作用）。
- 聚合：ClusterRole 的 `aggregationRule` 由其他 ClusterRole 的 label 组合权限。

**面试加分点**：
- 「`edit` 能改 Secret 吗？」能，但看不到 RBAC；`view` 能看所有资源包括 Secret（**这是常见的权限泄漏**）。
- 「pods/exec 与 pods/log 是独立资源」需要单独授权。
- 「RoleBinding 绑定 ClusterRole」是把集群级权限「投影」到 namespace。

### 73. ServiceAccount Token 的演进？

**要点**：从永久 Secret Token → 限时、受众绑定、自动轮换的 Projected Token。

**展开**：
- 旧模型（<1.24）：创建 SA 自动生成 Secret（`kubernetes.io/service-account-token`），Token 永久有效，挂载到 `/var/run/secrets/kubernetes.io/serviceaccount/token`。
- 新模型（1.24+ 默认）：`projected` volume，Token 由 kubelet 请求（`TokenRequest` API），有效期默认 1 小时，自动续期；带 `audience` 与 `boundObjectReference`（绑定 Pod），Pod 删除即失效。
- 需要长期 Token 的场景：CI/CD 外部系统、跨集群。用 `kubectl create token` 或手动创建 Secret（`type: kubernetes.io/service-account-token`，1.24 需要注解）。
- 安全风险：Token 泄漏 = 被冒充；用 `automountServiceAccountToken: false` 关闭不需要的挂载。

**面试加分点**：
- 「为什么 1.24 后有些应用挂了？」它们依赖 SA Token 存在的 Secret，或需要永久 Token。
- 「Token 的 audience 是什么？」用于与外部身份系统（如 AWS IRSA、GCP Workload Identity）做信任关系。
- 「如何限制 Token 的能力？」绑定最小权限的 SA + RBAC，不要默认用 `default` SA。

### 74. Pod Security Standards 与 PSA？

**要点**：内置三档策略（Privileged/Baseline/Restricted），通过 namespace label 启用，替代 PSP。

**展开**：
- 三档：
  - `privileged`：无限制（系统组件）。
  - `baseline`：禁止特权容器、hostPath 关键路径、hostNetwork/hostPID、危险 capability（`NET_ADMIN` 等需注意）、`allowPrivilegeEscalation` 默认允许但会限制。
  - `restricted`：`runAsNonRoot`、`readOnlyRootFilesystem`（可选）、drop ALL capabilities、seccomp `RuntimeDefault`、不允许 `hostPath`。
- 启用方式：namespace 标签
  ```yaml
  pod-security.kubernetes.io/enforce: restricted
  pod-security.kubernetes.io/audit: restricted
  pod-security.kubernetes.io/warn: restricted
  ```
- 版本标签（`pod-security.kubernetes.io/enforce-version: v1.30`）用于锁定策略版本。
- PSP 在 1.25 移除，PSA 是内置的准入逻辑，无需额外 controller。

**面试加分点**：
- 「如何迁移到 restricted？」先用 `warn` 观察，修镜像与 securityContext，再 `enforce`。
- 「为什么 Pod 建不出来，报 `violates PodSecurity`？」查看基线要求，常见是缺少 `seccompProfile` 或 `runAsNonRoot` 写了但镜像以 root 运行。
- 「PSA 能替代网络策略吗？」不能，它只管 Pod 的 securityContext。

### 75. 如何限制容器的权限？

**要点**：最小化 securityContext、drop capabilities、只读根文件系统、非 root、seccomp。

**展开**：
- Pod/Container `securityContext`：
  - `runAsNonRoot: true` + `runAsUser: 1000`
  - `readOnlyRootFilesystem: true`（需要可写目录用 emptyDir）
  - `allowPrivilegeEscalation: false`
  - `capabilities: { drop: ["ALL"], add: ["NET_BIND_SERVICE"] }`
  - `seccompProfile: { type: RuntimeDefault }`
  - `fsGroup` + `fsGroupChangePolicy` 控制卷权限
- 更外层：User Namespace（1.25 alpha → 1.33 beta，`hostUsers: false`）、AppArmor/SELinux、gVisor/Kata 沙箱、`RuntimeClass` 指定沙箱运行时。
- 限制手段：PSA `restricted`、Kyverno/Gatekeeper 策略（禁止 `privileged`、`hostPath`、`latest` 标签）。
- 镜像：非 root 基础镜像（distroless）、去掉 shell。

**面试加分点**：
- 「readOnlyRootFilesystem 后应用写 /tmp 失败？」挂一个 emptyDir 到 /tmp。
- 「drop ALL 后 bind 80 端口失败？」非 root 不能绑定 <1024，改用 8080 + Service 映射，或加 `NET_BIND_SERVICE`。
- 「seccomp 会破坏应用吗？」某些 JIT/GPU 应用需要特定 syscall，需用 `Localhost` 自定义 profile 调试。

### 76. 为什么不要用 `privileged: true`？

**要点**：等价于宿主机 root，容器逃逸风险极高。

**展开**：
- privileged 会：获得所有 capabilities、关闭 seccomp/AppArmor 默认限制、访问所有主机设备、绕过 cgroup 的部分限制，可直接挂载宿主文件系统。
- 典型逃逸：`privileged` + `hostPath: /` → 写宿主 crontab / 修改 kubelet 配置 → 控制整个节点。
- 替代方案：
  - 需要某个 capability（如 `NET_ADMIN`）→ 只加这一个。
  - 需要设备（GPU/TPU）→ device plugin + `resources.limits`。
  - 需要主机网络（如 CNI）→ 显式 `hostNetwork` + 专用节点。
  - 需要加载内核模块 → 能用 DaemonSet + 专用节点尽量避免。
- 治理：PSA `baseline` 已禁止 privileged（除非用 `privileged` namespace），Kyverno 全局策略。

**面试加分点**：
- 「必须用 privileged 的场景？」本地调试、某些 CNI/存储驱动（如某些 CSI node 插件需要 `mount` + privileged），但仍应限定到专用 namespace + 节点。
- 「如何检测现有 privileged Pod？」`kubectl get pods -A -o json | jq '...[].spec.containers[].securityContext.privileged'`。
- 「RuntimeClass 能替代吗？」gVisor/Kata 可以在不 privileged 的前提下提供更强隔离，但兼容性有代价。

### 77. 网络与密钥的安全实践？

**要点**：默认拒绝、加密、最小权限、审计、镜像可信。

**展开**：
- 网络：NetworkPolicy 默认拒绝 + 白名单；控制面只对内网开放；API Server 关闭匿名访问；Ingress 只暴露必要端口；DNS 也要在 egress 白名单里。
- 密钥：etcd 静态加密（KMS provider）、外部密钥管理（Vault/AWS SM）、Secret 不进 Git、CI 用短期凭据、定期轮换。
- 身份：RBAC 最小权限、OIDC 集成、禁用长期 SA Token、`automountServiceAccountToken: false`。
- 节点：kubelet `--anonymous-auth=false`、`--authorization-mode=Webhook`、`readOnlyPort: 0`、限制 SSH。
- 审计：`audit-policy.yaml`、集中到 SIEM、告警（如 `secrets` 的 get 操作）。
- 供应链：镜像签名（cosign）、扫描（Trivy）、准入校验（Kyverno `verifyImages`）、SBOM。

**面试加分点**：
- 「etcd 加密后旧数据怎么办？」`kube-apiserver` 有 `--encryption-provider-config-automatic-reload`，需要 `kubectl get secrets -A -o json | kubectl replace` 重写才能加密存量数据。
- 「如何检测异常？」审计日志 + 运行时安全（Falco）。
- 「控制面暴露公网的风险」优先私有端点 + VPN/Bastion。

### 78. 供应链安全在 K8s 里怎么做？

**要点**：来源可信、内容可扫描、产物可验证、部署可拦截。

**展开**：
- 构建：最小基础镜像（distroless/alpine）、固定依赖版本、SBOM（Syft）、漏洞扫描（Trivy/Grype）、SBOM 与镜像一起存。
- 签名：cosign（keyless 或 KMS）、Sigstore，签名后推送到私有 registry。
- 准入校验：Kyverno `verifyImages`、Sigstore policy-controller、Gatekeeper + Ratify，拒绝未签名/来源不明的镜像。
- 运行时：只允许私有仓库（admission 策略限制 `image` 前缀）、禁用 `latest`、`imagePullPolicy: Always` 配合 digest 固定。
- 供应链攻击面：CI 凭据、构建缓存投毒、依赖混淆、镜像仓库权限。

**面试加分点**：
- 「如何保证部署的是签过名的镜像？」准入 webhook 校验 cosign 签名 + 镜像 digest。
- 「SBOM 有什么用？」漏洞响应时快速定位受影响的镜像。
- 「CI 与 CD 的信任边界」CI 只推镜像，CD 只部署已签名产物。

### 79. Admission Webhook 的 Mutating 与 Validating 区别？

**要点**：Mutating 先执行、可改对象；Validating 后执行、只判断。

**展开**：
- 顺序：Mutating（可多个，按字母序）→ 内置校验 → Validating（可多个）→ 持久化。
- Mutating 常见用途：注入 sidecar（Istio）、补默认值、加 label/annotation、修改镜像地址。
- Validating 常见用途：拒绝无 resource limits 的 Pod、拒绝 `latest` 标签、校验 Ingress host 唯一性、Kyverno/Gatekeeper 策略。
- 实现：`ValidatingWebhookConfiguration` / `MutatingWebhookConfiguration`，`clientConfig.service`（集群内）或 `url`（外部），`caBundle` 校验服务端证书。
- 可用性风险：
  - `failurePolicy: Fail`（默认）→ webhook 不可用时**拒绝所有匹配的请求**（可能瘫痪集群）。
  - `failurePolicy: Ignore` → 放过（安全策略可能被绕过）。
  - `timeoutSeconds`（默认 10s，最大 30s）过长会拖慢 apiserver。
  - `namespaceSelector`/`objectSelector` 要排除 webhook 自身的 namespace，避免死锁。

**面试加分点**：
- 「webhook 把自己锁死了怎么办？」用 `namespaceSelector` 排除 `kube-system` 与 webhook 所在 ns；紧急时删除 `*WebhookConfiguration`。
- 「Mutating 如何幂等？」重复执行不能产生叠加副作用（如重复加 sidecar）。
- 「性能影响？」每个创建/更新请求都会调用，长耗时会导致 apiserver 队列堆积。

### 80. 多租户隔离怎么做？

**要点**：按隔离强度分层：namespace 软隔离 → 节点池隔离 → 独立集群。

**展开**：
- **软多租户（namespace 级）**：
  - RBAC（每租户 Role/RoleBinding）。
  - ResourceQuota + LimitRange（防止资源挤占）。
  - NetworkPolicy（默认拒绝 + 租户内互通）。
  - PSA restricted（防特权容器）。
  - 存储配额（`resourcequotas` 的 `requests.storage`）。
  - Admission 策略限制 `hostPath`/`hostNetwork`/NodePort。
- **节点池隔离**：租户专用节点 + taint/toleration + nodeAffinity + 反亲和，防止与别人共享节点。
- **强隔离**：独立集群（最彻底）、vCluster（虚拟控制面）、Kata/gVisor（内核级隔离）、独立 etcd。
- 共享风险：内核漏洞、kubelet 逃逸、RBAC 配置错误、网络策略遗漏、资源争抢（CPU 节流）。

**面试加分点**：
- 「租户能创建 CRD 吗？」默认不能（cluster-scoped 资源要限制），否则可能影响全局。
- 「如何防租户提权？」禁止 `create/update` RBAC 对象、禁止 `escalate`/`bind` verb、禁止创建 privileged Pod。
- 「cost 归因？」按 namespace label 做成本分摊（Kubecost/OpenCost）。

### 81. ResourceQuota 与 LimitRange 的区别？

**要点**：Quota 限制「namespace 总量」，LimitRange 限制「单个 Pod/容器的默认与上下限」。

**展开**：
- **ResourceQuota**：
  - 计算资源：`requests.cpu`、`requests.memory`、`limits.cpu`、`limits.memory`。
  - 对象数量：`pods`、`services`、`configmaps`、`secrets`、`persistentvolumeclaims`、`services.loadbalancers`、`services.nodeports`。
  - 存储：`requests.storage`、`<sc-name>.storageclass.storage.k8s.io/requests.storage`。
  - 作用：创建超过配额的对象会被拒绝（`exceeded quota`）。
- **LimitRange**：
  - `default` / `defaultRequest`：未设置时的默认值。
  - `min`/`max`：单容器上下限。
  - `maxLimitRequestRatio`：limits/requests 比例上限。
  - `max`：对单个 Pod 总量。
- 两者常配合：Quota 控总量，LimitRange 兜默认值（避免没写 requests 的 Pod 绕过调度）。

**面试加分点**：
- 「配了 Quota 后 Pod 创建失败 `must specify limits.cpu`？」Quota 要求每个容器都声明 limits（当 Quota 包含 `limits.*` 时）。
- 「Quota 能限制 GPU 吗？」能，通过 `requests.nvidia.com/gpu` 这类扩展资源。
- 「LimitRange 只影响新建对象」，已有 Pod 不会被回溯修改。

### 82. 审计日志能回答什么问题？

**要点**：谁、何时、从哪、对什么、做了什么、结果如何；用于合规、排障、入侵检测。

**展开**：
- 审计级别：
  - `None`：不记录。
  - `Metadata`：只记录请求元数据（谁、什么资源、结果）。
  - `Request`：加上请求体（不含响应）。
  - `RequestResponse`：请求与响应都记（敏感，慎用）。
- 策略：`audit-policy.yaml` 按 `user`、`verb`、`resource`、`namespace` 定义规则；`omitStages` 跳过 `RequestReceived`。
- 输出：`--audit-log-path`（文件）、`--audit-webhook-config-file`（推送到 SIEM），**不要把审计日志写到 stdout**（滚动与性能问题）。
- 配置：`--audit-log-maxsize`、`--audit-log-maxbackup`、`--audit-log-maxage`。
- 典型查询：谁删了 namespace、谁 exec 进 Pod、谁读了 Secret、谁修改了 RBAC。

**面试加分点**：
- 「审计日志太大会拖垮 apiserver」——用 webhook 异步推送 + 只记录必要级别。
- 「如何检测异常？」规则告警：`secrets` 的 `get/list`、`pods/exec` 频繁、`cluster-admin` 绑定变化。
- 「与事件（Events）的区别」——Events 是对象状态变化，审计是 API 请求级别。

---

## 七、可观测性与运维（83–92）

### 83. 如何排查 CrashLoopBackOff？

**要点**：先看退出码和上一次日志，再分「应用自身」「配置依赖」「资源限制」「探针误杀」四类。

**展开**：
- 步骤：
  1. `kubectl describe pod` → 看 `Last State`、`Reason`、`Exit Code`、`Restart Count`。
  2. `kubectl logs <pod> --previous`（崩溃前的日志）、`kubectl logs <pod> -c <container>`。
  3. 退出码：
     - `0`：正常退出但 restartPolicy=Always → 应用不该退出（可能是启动脚本问题）。
     - `1`：应用错误（配置、连接失败、异常）。
     - `137`：SIGKILL，常见 OOMKilled 或 grace period 超时。
     - `143`：SIGTERM，被优雅终止。
     - `139`：段错误。
  4. 检查依赖：DB 地址、环境变量、Secret/ConfigMap 是否存在。
  5. 检查探针：liveness 太激进导致反复重启。
  6. 检查资源：内存 limit 太小、CPU throttling 导致启动超时。
- 工具：`kubectl debug`（临时容器）、`kubectl exec` 进容器手动跑启动命令。

**面试加分点**：
- 「CrashLoopBackOff 是错误还是状态？」是 kubelet 的重启退避状态，不是根因。
- 「OOMKilled 怎么确认？」`describe` 里有 `Last State: Terminated, Reason: OOMKilled`，或 `dmesg` 里 cgroup OOM。
- 「镜像本身启动就退出的排查？」本地 `docker run` 复现，检查 `ENTRYPOINT`/`CMD`。

### 84. Pod 一直 Pending 怎么查？

**要点**：看 Events 里的 `FailedScheduling` 原因，逐条排除。

**展开**：
- 常见原因：
  - `Insufficient cpu/memory`：资源不足 → 扩容节点或调小 requests。
  - `node(s) didn't match Pod's node affinity/selector`。
  - `node(s) had untolerated taint`。
  - `volume node affinity conflict` / `waiting for first consumer`。
  - `exceeded quota`。
  - `pod has unbound immediate PersistentVolumeClaims`。
  - `no nodes available to schedule pods`（调度器未运行或节点全部 NotReady）。
  - Pod 被 webhook 拒绝（不在 scheduling 阶段，但会表现为创建失败）。
  - `Preemption is not helpful`（抢占也无解）。
- 命令：
  - `kubectl describe pod` 的 Events。
  - `kubectl get events --field-selector involvedObject.name=<pod>`。
  - `kubectl get nodes -o wide` 看资源与状态。
  - 调度器日志 `kubectl logs -n kube-system -l component=kube-scheduler`。

**面试加分点**：
- 「Pending 与 ContainerCreating 的区别？」Pending 是还没调度/没开始创建；ContainerCreating 是已调度但创建卡住。
- 「大集群调度慢导致 Pending？」看调度器指标与 `percentageOfNodesToScore`。
- 「Pod 有 nodeName 但仍 Pending？」说明调度器跳过，kubelet 没接管或节点不健康。

### 85. 镜像拉取失败（ImagePullBackOff）的排查？

**要点**：镜像名/标签、认证、网络、仓库限流、架构不匹配。

**展开**：
- 错误信息解读：
  - `manifest unknown`：镜像或 tag 不存在。
  - `unauthorized`/`authentication required`：私有仓库凭据缺失。
  - `denied`：IAM/仓库权限不足。
  - `no such host`/`i/o timeout`：DNS 或网络问题。
  - `toomanyrequests`：Docker Hub 限流。
  - `no match for platform in manifest`：架构不匹配。
- 排查：
  1. `kubectl describe pod` 看具体错误。
  2. `imagePullSecrets` 是否正确挂到 Pod/SA。
  3. 节点上手动拉：`crictl pull <image>`。
  4. 仓库地址是否在 `imagePullPolicy` 生效范围（`Always`/`IfNotPresent`）。
  5. 多架构镜像是否包含节点架构。
  6. 代理/防火墙/私有 CA（`registries.conf` 的 `insecure` 或 `ca` 配置）。
- 私有仓库：`docker-registry` 类型的 Secret，或 kubelet 的 `config.json`。

**面试加分点**：
- 「Pod 用 SA 拉镜像」——把 `imagePullSecrets` 加到 ServiceAccount 上，所有 Pod 自动继承。
- 「拉取成功后运行失败？」可能是架构不兼容（`exec format error`）或镜像 entrypoint 问题。
- 「大规模并发拉取打爆 registry？」用镜像预热（DaemonSet 预拉）、registry 缓存、`IfNotPresent`。

### 86. 节点 NotReady 的常见原因？

**要点**：kubelet、容器运行时、网络、磁盘/内存压力、证书。

**展开**：
- 排查顺序：
  1. `kubectl describe node` 看 Conditions（`Ready`、`MemoryPressure`、`DiskPressure`、`PIDPressure`）与 Events。
  2. 节点上 `systemctl status kubelet`、`journalctl -u kubelet -n 100`。
  3. 运行时：`systemctl status containerd`、`crictl ps`。
  4. 网络：到 apiserver 的 6443 是否通、CNI Pod 是否正常。
  5. 磁盘：`df -h`、`/var/lib/kubelet`、`/var/lib/containerd` 是否满。
  6. 证书：kubelet 客户端证书是否过期（`openssl x509 -enddate`）。
  7. 系统：`dmesg` 看 OOM、内核 panic、磁盘故障。
- 自动机制：kubelet 每 10s 更新 Lease，node controller 在 `node-monitor-grace-period`（默认 40s）后标记 `Unknown/NotReady`，再按污点容忍度驱逐 Pod。

**面试加分点**：
- 「NotReady 后 Pod 会被立刻驱逐吗？」不会，默认 300s（`node.kubernetes.io/unreachable` 容忍 300s）。
- 「节点恢复后 Pod 会回来吗？」已驱逐的不会自动回来，由控制器重建；未驱逐的会继续运行。
- 「大量节点同时 NotReady？」可能是 apiserver/网络分区，而不是每个节点都坏了。

### 87. 内存泄漏和 OOM 怎么定位？

**要点**：区分「容器 OOM」与「节点 OOM」，再看是应用泄漏还是 limit 太小。

**展开**：
- 现象：`OOMKilled`（exit 137）、Pod 重启、节点 `MemoryPressure` 驱逐。
- 定位：
  1. `kubectl describe pod` 的 `Last State`。
  2. `kubectl top pod` 与 Prometheus `container_memory_working_set_bytes`（注意用 working_set，不是 RSS，更接近 OOM 判定）。
  3. 容器内 cgroup：`/sys/fs/cgroup/memory.max`、`memory.current`、`memory.events`（`oom` 计数）。
  4. 应用层：heap dump、pprof、`jmap`/`jcmd`、GC 日志。
  5. 节点层：`dmesg`、`journalctl -k`、`kubectl describe node`。
- 常见原因：JVM 堆外内存、连接池不释放、缓存无上限、goroutine/线程泄漏、sidecar 内存、page cache 计入？(cgroup v2 的 working_set 不含可回收 cache)。

**面试加分点**：
- 「limit 给多少合适？」根据 P95 峰值 + 20–30% 余量，配合监控长期观察；不要照抄 requests。
- 「OOMKilled 但内存没到 limit？」可能是 cgroup v1/v2 计数差异、sidecar 或 init 容器、或者内核 OOM 杀错了进程。
- 「如何预防？」压测、内存泄漏检测（持续监控趋势）、VPA 建议值、limit 与 requests 一致。

### 88. CPU throttling 怎么判断？

**要点**：看 CFS 节流比例，高节流说明 limit 太低或单线程/GC 导致尖峰。

**展开**：
- 指标：`container_cpu_cfs_throttled_periods_total / container_cpu_cfs_periods_total`（Prometheus），或容器内 `cpu.stat` 的 `nr_throttled`/`throttled_time`。
- 判断阈值：持续 > 5–10% 就值得关注；> 25% 通常影响明显（P99 延迟抬升）。
- 原因：
  - limit 设得太低（尤其 Java/Go GC 尖峰）。
  - requests 与 limit 差距太大（Burstable 下的突发被限）。
  - 应用线程模型：单核容器跑多线程。
  - 节点 CPU 超卖严重（requests 总和 > capacity）。
- 处理：
  - 提高 limit（或取消 CPU limit，只保留 requests，让调度保证基线）。
  - 调整 GC/并发参数。
  - 分片/水平扩展。

**面试加分点**：
- 「为什么去掉了 CPU limit？」很多人只设置 requests，避免节流导致的尾延迟；代价是可能挤占同节点其他 Pod（靠 requests 与 QoS 控制）。
- 「CPU 节流会导致重启吗？」不会，只会变慢；内存超限才会 OOMKill。
- 「如何验证是节流而非应用慢？」对比 `container_cpu_usage_seconds_total` 与 throttled 比例。

### 89. 集群监控一般监控什么？

**要点**：分层：节点 → 控制面 → 工作负载 → 业务；再加事件与日志。

**展开**：
- 节点：CPU/内存/磁盘 IO/网络/文件系统 inode、`node_load1`。
- 控制面：
  - apiserver：`apiserver_request_duration_seconds`、`apiserver_request_total{code}`、`apiserver_current_inflight_requests`、watch 数量。
  - etcd：fsync 延迟、DB 大小、leader 变化、wal 延迟。
  - scheduler：调度延迟、Pending Pod 数。
  - controller-manager：reconcile 耗时与错误。
- 工作负载：容器 CPU/内存/网络、重启次数、OOM、throttling、readiness 失败、HPA 状态。
- 网络：CoreDNS 延迟/错误、CNI 状态、Ingress 4xx/5xx 与 P99、kube-proxy 规则同步。
- 业务：SLO/SLI（可用性、延迟、错误率）、队列积压。
- 告警：以 SLO 为中心（burn rate），而不是单指标阈值；Alertmanager 路由与抑制。

**面试加分点**：
- 「用哪些工具？」Prometheus + Grafana + Alertmanager；日志 Loki/ELK；链路 OTel + Tempo/Jaeger；事件 kube-state-metrics（`kube_pod_container_status_restarts_total`）。
- 「kube-state-metrics 与 metrics-server 的区别？」前者是对象状态指标（副本数、重启、状态），后者是资源用量。
- 「为什么需要 SLO 而不是阈值告警？」减少噪音、直接反映用户体验。

### 90. Kubernetes 日志方案怎么设计？

**要点**：容器写 stdout/stderr → 节点日志文件 → DaemonSet 采集 → 集中存储；结构化 + 上下文。

**展开**：
- 链路：kubelet 写 `/var/log/pods/<ns>_<pod>_<uid>/<container>/0.log` → 采集器（Fluent Bit/Vector/Promtail/Filebeat）读 → 加元数据（pod、namespace、labels、node）→ 推送 Loki/ES/S3。
- 元数据：靠 CRI 日志格式解析，或 kubelet 的 `containerLogMaxSize`/`containerLogMaxFiles` 控制轮转。
- 采集策略：
  - 小规模：DaemonSet 采集所有 Pod。
  - 大规模：按 namespace/label 过滤，或 sidecar 采集特定文件。
  - 应用直接写 journald/文件 → 需要额外配置。
- 查询：Loki（低成本、label 索引）vs ES（全文检索、资源重）。
- 最佳实践：结构化 JSON、trace_id/request_id、避免多行堆栈（用 parser 合并）、不要把日志写到容器层可写层。

**面试加分点**：
- 「日志太多怎么办？」采样、级别控制、按 namespace 限流（Fluent Bit 的 `throttle`）。
- 「如何关联 trace？」统一 `trace_id` + OTel Collector。
- 「kubectl logs 看不到日志？」容器重启后旧日志在 `--previous`；节点日志被轮转删除；日志走 journald。

### 91. 集群升级流程？

**要点**：先控制面、再节点、注意版本偏差与废弃 API；小版本逐步升。

**展开**：
- 版本策略：kubeadm 只支持「小版本 ±1」升级（如 1.29 → 1.30 → 1.31），不能跳；kubelet 版本不得高于 apiserver（允许落后 2–3 个小版本）。
- 流程：
  1. 预检：`kubeadm upgrade plan`、废弃 API 扫描（`kubectl-api-deprecations`/Pluto）、CNI/CSI/Ingress 兼容性。
  2. 备份 etcd 与工作负载清单。
  3. 升级第一个控制面：`kubeadm upgrade apply v1.30.x`。
  4. 升级其余控制面：`kubeadm upgrade node`。
  5. 升级 kubelet/kubectl（apt/yum）。
  6. 逐节点 drain → 升级 → uncordon。
  7. 验证：Pod 状态、网络、存储、HPA、Ingress。
- 托管集群：控制面自动升级，节点池用 surge upgrade（新建节点再删旧的）。
- 风险控制：PDB、分批、灰度环境先验证、准备回滚（降级不支持，只能恢复 etcd 快照）。

**面试加分点**：
- 「升级后 Pod 起不来？」多为废弃 API（如 `policy/v1beta1` PSP）、`/var/lib/kubelet` 配置漂移、CNI 版本不兼容。
- 「kubelet 与 apiserver 版本差太多会怎样？」API 不兼容，可能导致节点注册失败。
- 「如何做到零中断？」多副本 + PDB + 足够 surge，且控制面本身多副本。

### 92. 如何安全地 drain 节点？

**要点**：先 cordon，再 drain；检查 PDB、优雅退出、数据安全。

**展开**：
- 命令：
  - `kubectl cordon <node>`：标记不可调度。
  - `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --grace-period=60 --timeout=300s`。
  - `kubectl uncordon <node>`：恢复。
- drain 会调用 eviction API，受 PDB 约束；违反时返回 429 并重试。
- 注意：
  - DaemonSet Pod 不会被驱逐（`--ignore-daemonsets`）。
  - `emptyDir` 数据会丢（`--delete-emptydir-data` 显式确认）。
  - StatefulSet Pod 迁移时卷会 detach/attach，注意 RWO 与 zone。
  - 长任务（Job）会被中断，需要 `--pod-selector` 或先暂停。
  - 驱逐顺序与 `terminationGracePeriodSeconds` 配合，避免截断请求。
- 观测：`kubectl get pods --field-selector spec.nodeName=<node>` 持续观察。

**面试加分点**：
- 「drain 卡住怎么办？」PDB 不允许、kubelet 不健康、Pod 有 finalizer、卷卸载失败。
- 「为什么先 cordon？」防止驱逐的同时又被调度新 Pod。
- 「如何做到无感？」多副本 + 反亲和 + PDB + preStop；单副本服务必然有中断。

---

## 八、故障排查实战（93–97）

### 93. 服务 502/503 的排查思路？

**要点**：先区分「网关到后端」还是「后端本身」，再看 Endpoints、端口、就绪、超时。

**展开**：
- **502 Bad Gateway**（网关收到了无效的后端响应）：
  - Ingress/网关到 Pod 的连接失败：Pod 未监听、端口不匹配（`targetPort` 错）。
  - 后端进程崩溃/重启中。
  - 协议不匹配（HTTP 后端用了 HTTPS，或 gRPC 未配 h2c）。
  - 超时设置不合理（`proxy-read-timeout` 太短，后端还在处理）。
  - Keepalive 与 `connection reset`（node 上 conntrack 或后端主动关闭）。
- **503 Service Unavailable**（没有可用后端）：
  - Endpoints 为空：selector 不匹配、Pod 未 Ready、Pod Terminating。
  - Controller 限流/熔断（nginx `limit_conn`、Istio circuit breaker）。
  - Service 不存在或被删（Ingress 引用错误）。
- 排查顺序：
  1. `kubectl get endpointslices -n <ns>` 看后端。
  2. `kubectl get pods -o wide` 看 Ready 与 IP。
  3. `kubectl describe ingress` 看后端 Service 与端口。
  4. Ingress Controller 日志（`kubectl logs -n ingress-nginx ...`）。
  5. 在 Pod 内 `curl localhost:<port>` 确认应用正常。
  6. 从另一 Pod `curl <svc>:<port>` 确认网络。
  7. NetworkPolicy 检查。

**面试加分点**：
- 「滚动更新时出现短暂 5xx」通常是 preStop/grace period/readiness 配置问题。
- 「只有部分请求 5xx」可能是单个 Pod 有问题、连接复用异常、或负载不均。
- 「502 与 504 的区别？」502 是后端返回无效响应/连接被拒；504 是后端超时。

### 94. 集群 DNS 解析失败的排查？

**要点**：从 Pod 的 resolv.conf → CoreDNS → 网络策略 → 上游，逐层验证。

**展开**：
- 步骤：
  1. Pod 内：`cat /etc/resolv.conf`（nameserver 是否为 kube-dns IP）、`nslookup kubernetes.default`、`dig @<coredns-ip> svc.ns.svc.cluster.local`。
  2. CoreDNS：`kubectl get pods -n kube-system -l k8s-app=kube-dns`、`kubectl logs`、`kubectl get svc kube-dns`。
  3. 到 CoreDNS 的连通：NetworkPolicy 是否放行 UDP/TCP 53。
  4. 节点层面：conntrack 表满、iptables 规则丢失、节点负载。
  5. 上游：`/etc/resolv.conf` 的 forward 配置、企业 DNS 可达性。
  6. 间歇性失败：NodeLocal DNSCache 未部署、CoreDNS 副本不足、UDP 丢包。
- 典型错误：
  - `server misbehaving`：上游 DNS 拒绝/超时。
  - `no such host`：解析不到（Service 不存在或 ndots 问题）。
  - `i/o timeout`：网络不通或 DNS 无响应。
- 临时排查：`kubectl run dnsutils --image=registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3 -- sleep 3600`。

**面试加分点**：
- 「为什么 Pod 内能解析 Kubernetes 默认域名但外部不行？」`forward` 上游配置或 egress 被阻断。
- 「NodeLocal DNSCache 的好处」：本地缓存减少跨节点查询、避免 conntrack 竞争、提高稳定性。
- 「CoreDNS 的高可用」：多副本 + 反亲和 + PDB + `topologySpread` 跨 zone。

### 95. etcd 出问题的表现与处理？

**要点**：控制面写操作变慢/失败、Leader 切换；根因多在磁盘 IO、网络、配额。

**展开**：
- 表现：
  - `kubectl` 报 `etcdserver: request timed out`、`context deadline exceeded`。
  - apiserver 返回 500/503，创建/更新失败，读可能仍可用（watch cache）。
  - 节点心跳丢失，被误判 NotReady。
  - Leader 频繁切换（`etcd_server_leader_changes_seen_total` 增长）。
- 检查：
  - `etcdctl endpoint health --endpoints=...`、`endpoint status`（DB 大小、leader、raftIndex）。
  - 日志：`journalctl -u etcd` 或容器日志，看 `slow fdatasync`、`failed to send out heartbeat`、`database space exceeded`。
  - 磁盘：`iostat -x 1`（await、%util）、`df`、`fio` 基准。
  - 网络：`ping`、`etcdctl check perf`。
  - 证书：过期会导致 peer 连接失败。
- 处理：
  - 空间满：`etcdctl compact` + `defrag`（逐节点，会阻塞），调大 `--quota-backend-bytes`。
  - 磁盘慢：换 SSD、隔离 IO、减少其他写入。
  - 成员故障：移除并重新加入（`member remove`/`add`），避免双主。
  - 灾备：`snapshot save` + `snapshot restore`（新集群）。
- 预防：独立磁盘、定期 defrag/compact、监控 fsync 延迟、限制对象大小与写入频率。

**面试加分点**：
- 「etcd 挂了集群会怎样？」已运行的 Pod 不受影响（kubelet 有自己的缓存），但不能新建/更新；节点心跳更新失败会导致误驱逐。
- 「为什么不能随便重启 etcd？」可能触发新的选举，频繁选举影响可用性；滚动重启时一次只动一个。
- 「恢复后 revision 变化」需要客户端重新 watch。

### 96. 如何排查「Pod 频繁被驱逐」？

**要点**：看节点 Condition 与 kubelet 驱逐阈值，再判断是资源压力还是人为。

**展开**：
- 现象：Pod 状态 `Evicted`，`describe` 的 `Status.Reason` 与 `Message` 会写明原因（如 `The node was low on resource: memory`）。
- 节点侧：
  - `kubectl describe node` 看 Conditions（`MemoryPressure`、`DiskPressure`、`PIDPressure`）。
  - kubelet 驱逐阈值：`--eviction-hard=memory.available<100Mi,nodefs.available<10%`、`--eviction-soft` + `--eviction-soft-grace-period`。
  - 看 `/var/lib/kubelet` 日志中的 `eviction manager` 记录。
- 常见原因：
  - 节点内存耗尽（应用泄漏、超卖、page cache）。
  - 磁盘写满（镜像、日志、`emptyDir`）。
  - inode 耗尽。
  - `ephemeral-storage` 超限（容器可写层）。
  - `PIDPressure`（进程数过多）。
- 人为原因：`kubectl drain`、节点维护、HPA 缩容、`descheduler`。
- 处理：给 Pod 设合理 requests/limits、`ephemeral-storage` limit、日志轮转、清理镜像、扩容节点、用 VPA 调参。

**面试加分点**：
- 「驱逐与 OOMKilled 的区别？」驱逐是 kubelet 主动按优先级杀 Pod；OOMKilled 是容器超自己的 limit 被内核杀。
- 「如何防止核心服务被驱逐？」Guaranteed QoS + `system-node-critical` 优先级 + 合理 limit。
- 「磁盘压力下谁先被驱逐？」按 QoS 与 `ephemeral-storage` 使用量排序，BestEffort 最先。

### 97. apiserver 高延迟 / 429 怎么处理？

**要点**：区分「客户端行为」「apiserver 自身」「etcd 瓶颈」三类。

**展开**：
- 症状：`kubectl` 变慢、控制器报 `TooManyRequests`/`context deadline exceeded`、`apiserver_request_duration_seconds` P99 上升。
- 原因：
  - 客户端：list-watch 未用 `resourceVersion` 缓存，频繁全量 LIST；QPS 太高（client-go 默认 5 QPS，可 `--kube-api-qps` 调整）。
  - 大对象/大列表：巨大 ConfigMap/CRD、`kubectl get pods -A` 不带 limit。
  - 高频写：频繁更新 status/annotation、Lease 心跳过大、大量 event。
  - webhook 慢：admission 超时 10s。
  - etcd 慢：fsync 延迟、DB 过大。
  - 网络：LB 超时、TLS 握手。
- 处理：
  - **APF（API Priority and Fairness）**：按 FlowSchema 分优先级，防止一个客户端打爆。
  - 增加 apiserver 副本、增大 `--max-requests-inflight`（谨慎）。
  - 优化客户端：informers、`resourceVersion=0`（允许缓存读）、分页、`fieldSelector`。
  - 减少大对象与高频更新；event 使用 `--event-ttl` 与 `EventSeries` 聚合。
  - 优化 webhook：加缓存、缩短超时、限制匹配范围。
- 监控：`apiserver_request_total{code="429"}`、`apiserver_flowcontrol_rejected_requests_total`、`etcd_request_duration_seconds`。

**面试加分点**：
- 「429 是限流而不是故障」——要看是哪个 FlowSchema 被限。
- 「为什么 kubectl 比控制器慢？」kubectl 每次拿最新数据，控制器用 watch cache。
- 「apiserver 扩容有用吗？」读请求有（watch cache），写请求受 etcd 制约。

---

## 九、生产实践与进阶（98–100）

### 98. 生产集群的设计要点有哪些？

**要点**：可用性、隔离、安全、可观测、成本、演进性六个维度。

**展开**：
- **可用性**：
  - 控制面多副本 + 跨 AZ（kubeadm HA 或托管集群）。
  - etcd 3/5 节点 + 独立 SSD。
  - 节点池至少 2–3 个 AZ，Pod 用 topologySpread 打散。
  - PDB + 多副本 + 反亲和。
- **隔离**：namespace + RBAC + ResourceQuota + NetworkPolicy + PSA；专用节点池（taint）隔离关键业务。
- **安全**：镜像签名与准入、Secret 外部化、审计日志、最小权限、私有 API endpoint。
- **可观测**：Prometheus 高可用（Thanos/Mimir）、日志集中、OTel 链路、SLO 告警、事件与成本监控。
- **成本**：Karpenter/Cluster Autoscaler、Spot/竞价实例、VPA、`MostAllocated` 调度、闲置 namespace 清理。
- **演进**：GitOps（Argo CD）、渐进式发布（Rollouts）、预发环境、升级窗口、演练（混沌工程、故障注入）。
- **备份与容灾**：etcd 快照、Velero、跨区域备份、恢复演练（RTO/RPO）。
- **容量规划**：用 requests 总量 vs allocatable、监控 Pending 与调度失败率、预留 buffer。

**面试加分点**：
- 「托管 vs 自建？」托管省控制面运维、升级与 SLA，但定制性受限、成本结构不同；自建灵活但需要专职团队。
- 「多集群策略」：按环境/地域/租户拆分，用 Fleet/Karmada/Cluster API 管理。
- 「如何做灾难恢复？」明确 RPO/RTO，准备 runbook，定期演练（比备份本身更重要）。

### 99. GitOps 与传统 CI/CD 推模型有什么区别？

**要点**：Git 是唯一真相，集群内 Agent 拉取并持续调和，而不是 CI 直接推送。

**展开**：
- 推模型（传统）：
  - CI 构建 → 调 `kubectl apply`/`helm upgrade` 推到集群。
  - 问题：需要集群凭据在 CI 中、漂移不可见、回滚靠脚本、审计分散。
- 拉模型（GitOps）：
  - Git 存声明式清单 → Argo CD/Flux 在集群内 watch Git → 对比集群实际状态 → 自动 apply（或提示）。
  - 优点：凭据只存在集群内（最小暴露）、漂移检测（`OutOfSync`）、审计在 Git、回滚 = revert commit、多集群统一管理。
- 分工：CI 负责构建镜像、推仓库、更新 Git 中的镜像 tag/digest；CD（GitOps）负责部署。
- 关键实践：
  - 环境分离：目录/overlay/Helm values per env，而不是改同一份文件。
  - 镜像用 digest 固定（不可变）。
  - Secret 管理：Sealed Secrets、SOPS + age、External Secrets，不要明文进 Git。
  - 审批：PR + 环境分支/Argo CD 的 sync window。
  - 渐进式：Argo Rollouts + analysis。

**面试加分点**：
- 「GitOps 的缺点？」需要额外组件、Git 成为关键依赖、大规模 repo 性能、密钥管理复杂。
- 「如何处理集群专属配置？」用 Kustomize overlay 或 `ApplicationSet` 按集群参数化；不要硬编码环境差异。
- 「漂移了怎么办？」Argo CD 的 `selfHeal`、告警、`kubectl diff`。

### 100. 如何为大规模集群做扩展性优化？

**要点**：控制面（apiserver/etcd/scheduler）→ 数据面（kubelet/网络）→ 工作负载与可观测性，逐层优化。

**展开**：
- **控制面**：
  - APF 限流、限制客户端 QPS、减少大对象与高频 status 写入。
  - etcd 独立 SSD、定期 defrag/compact、控制 DB 大小。
  - apiserver 横向扩展 + watch cache；scheduler 用 `percentageOfNodesToScore` 与 profile。
  - 减少 controller 的 LIST 压力（informer 缓存）。
- **数据面**：
  - 网络：Cilium eBPF 替代 kube-proxy、EndpointSlice、拓扑感知、NodeLocal DNSCache。
  - 节点：NodeLocal 镜像缓存、合理的 `maxPods`、`kubeReserved`/`systemReserved` 预留、大页与 NUMA（HPC）。
  - 运行时：containerd 的镜像存储与 GC 调优、`serializeImagePulls: false`。
- **工作负载**：
  - 缩短启动（小镜像、预热、`startupProbe`）、合理 HPA/VPA、避免频繁滚动。
  - 减少高基数 label/annotation（增加 etcd 写入）。
  - 大 namespace 拆分、限制 object 数量。
- **可观测性**：
  - 指标降基数（不要用 Pod 名做 label）、采样、分层存储（Thanos/Mimir）。
  - 事件与日志聚合、核心指标优先。
- **运维**：分片（多集群/多控制面）、按域拆分、容量规划与压测（kubemark/kwok）。

**面试加分点**：
- 「K8s 官方的大规模目标？」单集群 5000 节点、15 万 Pod（大规模基准），但多数生产集群远大于此前仍会遇到瓶颈，需要针对性调优。
- 「瓶颈通常在哪？」etcd 写入、apiserver 请求放大、网络规则规模、调度器过滤性能。
- 「如何量化？」用 SLO 指标（调度延迟、API 延迟、Pod 启动时间）+ 压测，不要凭感觉。

---

## 附：高频追问速记

| 主题 | 一句话要点 |
| --- | --- |
| 为什么 Pod 是原子单位 | 共享网络与卷，强耦合容器一起调度 |
| 控制器模式 | watch 期望状态 → reconcile 实际状态 |
| 证书轮换 | kubeadm certs renew / 托管集群自动 |
| 优雅下线 | SIGTERM + preStop + 足够 grace period |
| 有状态服务 | StatefulSet + PVC 模板 + Operator |
| 网络隔离 | NetworkPolicy，需要 CNI 支持 |
| 权限最小化 | RBAC + PSA + drop capabilities |
| 发布策略 | 滚动 / 蓝绿 / 金丝雀 / 灰度，配合 PDB |
| 蓝绿 vs 灰度 | 蓝绿=全切快回滚但资源翻倍；灰度为总称，金丝雀是其子集 |
| 观测三件套 | Metrics（Prometheus）、Logs（Loki/ES）、Traces（OTel） |
| 灾难恢复 | etcd 快照 + Velero + 应用级备份 + 恢复演练 |
| KCM vs CCM | 集群内状态调和 vs 云资源适配层 |
| CRD vs Operator | 类型扩展 vs 类型 + 调谐逻辑 |
| Service 没 Endpoints | selector/Ready/namespace/端口，四查 |
| Pod Pending | 看 FailedScheduling：资源/亲和/污点/卷/配额 |
| CrashLoopBackOff | 退出码 + --previous 日志 + 探针 + 资源 |
| 502 vs 503 | 后端连接异常 vs 无可用后端 |
| 大规模优化 | 控制面限流、eBPF、拓扑感知、指标降基数 |
| kube-proxy 部署 | 每节点一个 DaemonSet；eBPF 替代后仍是每节点一个 Agent |
| DNAT | 改目标地址；只处理 TCP/UDP，ICMP 不通 |
| iptables vs IPVS | 规则链表逐条匹配 vs 哈希表直查；差在更新成本与调度算法 |
| EndpointSlice 拆分 | 每片最多 100 个 endpoint；按 addressType/端口分，**不按 zone** |
| RuntimeClass | 按 Pod 选运行时；`handler` 名必须与节点 CRI 配置一致 |
