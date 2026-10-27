# Envoy 与 Istio 学习索引

原文档已拆分为两份详解：

| 文档 | 内容 | 适合 |
| --- | --- | --- |
| [envoy.md](./envoy.md) | Envoy 进程模型、Listener/Filter/Route/Cluster、xDS 协议族、负载均衡/熔断/限流/TLS、可观测性、排障、与 Nginx 对比、面试题 | 想搞懂数据面、写 Envoy 配置、看 sidecar 指标 |
| [istio.md](./istio.md) | 服务网格动机、istiod 架构演进、iptables 劫持、核心 CRD、mTLS 与授权、Ambient 模式、请求全链路、安装升级、排障手册、面试题 | 想设计/运维服务网格、做流量治理与安全策略 |
| [istio-envoy-lab.md](./istio-envoy-lab.md) | 基于 kind + bookinfo 的 15 个动手实验：灰度、超时重试、熔断、故障注入、镜像、mTLS、JWT、可观测性、ServiceEntry、Ambient、排障练习 | 想真正跑一遍、面试前练手 |

## 建议阅读顺序

1. `envoy.md` 第一～三章：建立「代理如何转发流量」的心智模型。
2. `istio.md` 第三～五章：理解控制面从哪里来、配置长什么样。
3. `envoy.md` 第四章 xDS：把两者连起来（istiod 下发、Envoy ACK/NACK）。
4. `istio.md` 第九章「一次请求的完整旅程」：把前面所有概念串成一条线。
5. `istio-envoy-lab.md` 实验 0～7：动手验证灰度、超时、熔断。
6. `istio-envoy-lab.md` 实验 10～15：mTLS、授权、可观测性与排障。
7. 两份文档的排障章节对照看：现象往往横跨控制面与数据面。

## 实验前置要求

- Docker Desktop / Docker Engine（kind 需要）
- 至少 4 vCPU、8 GB 内存（demo profile 会装 Kiali/Jaeger/Grafana/Prometheus）
- 能访问 `registry.k8s.io`、`docker.io`、`ghcr.io`（bookinfo 镜像与 istio 组件）
- 网络受限时可提前 `docker pull` 镜像，或使用离线发行包

## 一句话对照

- **Envoy**：会转发、会均衡、会熔断的「引擎」，只会听 xDS 的指挥。
- **Istio**：编写并下发指挥指令的「大脑」，自己不转发任何字节。
