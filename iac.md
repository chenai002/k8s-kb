# IaC（基础设施即代码）知识梳理

> 从「为什么需要 IaC」讲到「Terraform / Pulumi / Ansible / GitOps 怎么选」，最后落到 K8s 场景下的边界划分。
> 面向：会用 K8s、想搞清楚基础设施该怎么管的人。

---

## 目录

- [一、一句话理解 IaC](#一一句话理解iac)
- [二、没有 IaC 之前是什么样](#二没有iac-之前是什么样)
- [三、IaC 到底在管什么](#三iac-到底在管什么)
- [四、两大流派：声明式 vs 命令式](#四两大流派声明式-vs-命令式)
- [五、核心概念（以 Terraform 为例）](#五核心概念以terraform-为例)
- [六、主流工具对比](#六主流工具对比)
- [七、状态（State）为什么是重点](#七状态state为什么是重点)
- [八、IaC 与 GitOps、CI/CD 的关系](#八iac-与-gitops-cicd-的关系)
- [九、在 Kubernetes 场景下怎么划边界](#九在-kubernetes-场景下怎么划边界)
- [十、实战中的坑](#十实战中的坑)
- [十一、面试怎么答](#十一面试怎么答)
- [附：速查表](#附速查表)

---

## 一、一句话理解 IaC

> **用代码（配置文件）来描述和管理基础设施，而不是点控制台或敲命令。**

```
传统方式：登录云控制台 → 点"创建 VPC" → 点"创建子网" → 点"创建 VM"
IaC 方式：写一个 .tf 文件 → terraform apply → 全部建好
```

**核心价值**：**可重复、可版本控制、可审查、可回滚**。

---

## 二、没有 IaC 之前是什么样

### 2.1 「雪花服务器」

```
服务器 A：2019 年建的，手工装过 nginx 1.18，改过 3 次配置
服务器 B：2020 年建的，nginx 1.20，配置文件和 A 不一样
服务器 C：谁来着？没人知道它上面跑了什么
```

每台机器都是独一无二的「雪花」，**不敢重启、不敢重建**。

### 2.2 典型痛点

| 痛点 | 后果 |
| --- | --- |
| **环境不一致** | 测试环境能跑、生产跑不了 |
| **无法重建** | 机器挂了只能手工救，救不回来就丢服务 |
| **变更靠记忆** | 谁在什么时候改了什么，没人知道 |
| **无法审查** | 生产变更没有 code review |
| **扩容靠手工** | 开 10 台机器，重复操作 10 次，每次都可能出错 |
| **文档过时** | 架构文档写完就过期，真实状态只在某个人的脑子里 |

**IaC 的答案**：**用代码记录真实状态，代码即文档，代码即可重建的依据**。

---

## 三、IaC 到底在管什么

按层次分，IaC 通常分三层，覆盖不同粒度的资源：

```
┌─────────────────────────────────────────┐
│ ③ 应用层（Application）                   │
│    Deployment / Service / ConfigMap      │
│    → GitOps（Argo CD / Flux）或 Helm     │
├─────────────────────────────────────────┤
│ ② 平台层（Platform）                      │
│    K8s 集群、节点池、Ingress、监控栈       │
│    → Terraform / Cluster API             │
├─────────────────────────────────────────┤
│ ① 基础设施层（Infrastructure）             │
│    VPC、子网、安全组、VM、负载均衡、DNS     │
│    → Terraform / Pulumi / 云厂商模板       │
└─────────────────────────────────────────┘
```

**关键认知**：
- **不是所有层都用同一个工具**。常见组合是「Terraform 管 ① 和 ②，Argo CD 管 ③」。
- 层次越高，变更越频繁，越需要「快速对账」（GitOps）。
- 层次越低，变更越少、越需要「精确的变更计划」（Terraform plan）。

### 3.1 除了「资源」，还在管什么

| 对象 | 用什么 |
| --- | --- |
| 云资源（VPC/VM/LB） | Terraform / Pulumi / CloudFormation |
| K8s 资源 | GitOps（Argo CD/Flux）、Helm、Kustomize |
| 配置管理（包、服务、文件） | Ansible、Chef、Puppet |
| 策略与合规 | OPA/Gatekeeper、Kyverno、Checkov/tfsec |
| 密钥 | Vault、SOPS、Sealed Secrets |

---

## 四、两大流派：声明式 vs 命令式

### 4.1 声明式（Declarative）

**描述「最终要什么」，不描述「怎么做」。**

```hcl
# Terraform：我要一个 3 个实例的集群，每个 t3.medium
resource "aws_instance" "web" {
  count         = 3
  instance_type = "t3.medium"
  ami           = "ami-12345678"
}
```

- 你改 `count = 5`，工具自己算出「要新建 2 台」。
- **幂等**：跑 1 次和跑 100 次结果一样。

**代表**：Terraform、Pulumi、CloudFormation、K8s 的 YAML、Ansible（大部分模块也是声明式）。

### 4.2 命令式（Imperative）

**描述「一步步做什么」。**

```bash
# 脚本式：明确写清楚顺序
aws ec2 run-instances --count 3 --instance-type t3.medium ...
aws elbv2 create-load-balancer ...
aws elbv2 register-targets ...
```

- 需要自己处理「已经存在怎么办」「失败重试」「依赖顺序」。
- **不幂等**：重复执行可能报错或重复创建。

**代表**：Shell 脚本、Ansible 的 `command`/`shell` 模块、云 CLI。

### 4.3 对比

| 维度 | 声明式 | 命令式 |
| --- | --- | --- |
| 描述什么 | 目标状态 | 执行步骤 |
| 幂等性 | ✅ 天然幂等 | ❌ 需要自己做 |
| 变更预览 | ✅ 可以 diff（`terraform plan`） | ❌ 无法预知 |
| 处理漂移 | ✅ 自动发现并纠正 | ❌ 不知道实际状态 |
| 灵活性 | 受限于工具支持的资源 | 无限（能做任何事） |
| 学习曲线 | 需要理解工具模型 | 熟悉 shell 即可 |
| 适合 | 资源创建与生命周期管理 | 一次性操作、复杂编排 |

> **实践建议**：**核心资源用声明式，边缘操作可以用命令式**。不要为了"纯"而硬用声明式。

---

## 五、核心概念（以 Terraform 为例）

### 5.1 最小结构

```hcl
# main.tf
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" {
  region = "ap-east-1"
}

resource "aws_s3_bucket" "logs" {
  bucket = "my-logs-bucket"
  tags = {
    Environment = "prod"
    Owner       = "platform-team"
  }
}
```

执行流程：

```bash
terraform init    # 下载 provider、初始化后端
terraform plan    # 显示"将要做什么"（不执行）
terraform apply   # 执行
terraform destroy # 销毁
```

### 5.2 四个核心概念

| 概念 | 作用 | 说明 |
| --- | --- | --- |
| **Provider** | 与某个 API 对话的插件 | `aws`、`google`、`kubernetes`、`helm`、`cloudflare` |
| **Resource** | 一个被管理的对象 | `aws_instance`、`kubernetes_deployment` |
| **Data Source** | 读取已有信息（只读） | `data "aws_ami" "ubuntu"` |
| **State** | 记录「我管理了什么」 | 存对象 ID、属性、依赖关系 |

### 5.3 模块化

```hcl
# 调用一个模块
module "vpc" {
  source = "./modules/vpc"
  cidr   = "10.0.0.0/16"
  azs    = ["ap-east-1a", "ap-east-1b"]
}

# 使用模块输出
resource "aws_instance" "app" {
  subnet_id = module.vpc.private_subnet_ids[0]
}
```

**目的**：复用、分层（网络层 / 计算层 / 数据层）、隔离变更影响。

### 5.4 变量与输出

```hcl
variable "env" {
  type    = string
  default = "dev"
}

output "bucket_name" {
  value = aws_s3_bucket.logs.bucket
}
```

- 变量：让同一份代码适配多环境（配合 `.tfvars`）。
- 输出：让模块之间传递信息，或给 CI/CD 使用。

---

## 六、主流工具对比

| 工具 | 类型 | 语言 | 特点 | 适合 |
| --- | --- | --- | --- | --- |
| **Terraform** | 声明式 | HCL | 生态最全、状态管理成熟、支持多云 | **默认选择**，绝大多数场景 |
| **OpenTofu** | 声明式 | HCL | Terraform 的开源分支（因 Terraform 改 BSL 许可） | 需要完全开源许可时 |
| **Pulumi** | 声明式 | TypeScript/Python/Go | 用真编程语言写 IaC，可复用库和测试 | 团队有强编程背景、需要复杂逻辑 |
| **AWS CloudFormation** | 声明式 | YAML/JSON | AWS 原生、深度集成、无需额外状态 | 纯 AWS、追求托管 |
| **CDK / CDKTF** | 声明式 | 语言编译成 CFN/TF | 用编程语言生成模板 | 希望在原工具上加编程能力 |
| **Ansible** | 偏命令式（也有声明模块） | YAML | 无 Agent、擅长配置管理与一次性任务 | 服务器初始化、配置下发、批量运维 |
| **Chef / Puppet** | 声明式 | Ruby/DSL | Agent 模式、成熟但重 | 传统企业环境 |
| **Helm** | 模板 + 声明 | YAML + Go template | K8s 应用打包 | 应用分发 |
| **Kustomize** | 声明式补丁 | YAML | 无模板，环境差异叠加 | 多环境差异 |
| **Argo CD / Flux** | 声明式（GitOps） | YAML | 持续对账，检测漂移 | K8s 应用交付 |
| **Crossplane** | 声明式（K8s CRD） | YAML | 用 K8s 方式管云资源 | 已深度使用 K8s 的团队 |

### 6.1 怎么选（决策树）

```
你要管什么？
│
├─ 云资源（VPC/VM/LB/数据库）
│   ├─ 多朵云 / 生态复杂 ──────────▶ Terraform（或 OpenTofu）
│   ├─ 纯 AWS + 想托管 ───────────▶ CloudFormation
│   └─ 团队强编程能力 ────────────▶ Pulumi
│
├─ 服务器上的软件与配置
│   └─▶ Ansible（无 agent、上手快）
│
├─ K8s 应用
│   ├─ 需要多环境差异 ────────────▶ Kustomize（或 Helm values）
│   ├─ 需要打包分发 ──────────────▶ Helm
│   └─ 需要持续对账 + 漂移检测 ────▶ Argo CD / Flux
│
└─ 想用 K8s 方式统一管云资源
    └─▶ Crossplane
```

### 6.2 Terraform vs Pulumi（常被问）

| | Terraform | Pulumi |
| --- | --- | --- |
| 语言 | HCL（专用 DSL） | TS/Python/Go/.NET |
| 学习成本 | 低（语法简单） | 中（要会编程 + 学 Pulumi API） |
| 逻辑能力 | 有限（`for_each`、`count`、条件） | 完整编程能力（循环、函数、类） |
| 测试 | 较难 | 单元测试友好 |
| 生态 | **最丰富** | 复用 Terraform provider（较完整） |
| 状态管理 | 成熟 | 类似 |

**结论**：**团队没有强编程需求，优先 Terraform**；需要大量自定义逻辑时才用 Pulumi。

---

## 七、状态（State）为什么是重点

### 7.1 状态是干什么的

Terraform 不是每次去云上扫描所有资源。它靠一份 **state 文件**记录：

```json
{
  "resources": [
    {
      "type": "aws_instance",
      "name": "web",
      "instances": [{ "attributes": { "id": "i-0abc123", "instance_type": "t3.medium" } }]
    }
  ]
}
```

**三方关系**：

```
代码（期望）  ←→  State（记忆）  ←→  云上真实资源
```

`terraform plan` 就是拿「代码」和「State + 刷新后的真实状态」做 diff。

### 7.2 状态存在哪

| 方式 | 适合 | 说明 |
| --- | --- | --- |
| **本地文件**（默认） | 学习、单人 | `terraform.tfstate`，**不要提交到 Git** |
| **远程后端**（S3+GCS+DynamoDB） | 团队 | 加锁防止并发修改 |
| **Terraform Cloud / OpenTofu 云** | 团队 | 托管状态，带 UI 与审计 |

**远程后端示例**：

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tfstate"
    key            = "prod/network.tfstate"
    region         = "ap-east-1"
    dynamodb_table = "tf-locks"     # 状态锁
    encrypt        = true
  }
}
```

### 7.3 状态带来的坑

| 坑 | 说明 | 应对 |
| --- | --- | --- |
| **状态里的资源和云上不一致** | 有人手动改了资源 | `terraform refresh` / `plan` 会显示漂移 |
| **并发修改** | 两个人同时 apply | 远程后端 + 状态锁 |
| **误删** | apply 把不该删的删了 | `plan` 必看、保护关键资源（`lifecycle.prevent_destroy`） |
| **状态泄漏敏感信息** | 状态里有密码 | 加密存储、最小权限访问 |
| **资源被手动删了** | 代码里还有 | `plan` 显示重建；用 `terraform state rm` 清理记忆 |

> **一句话**：**State 是 Terraform 的"记忆"，丢了或乱了，就等于管理失控。**

---

## 八、IaC 与 GitOps、CI/CD 的关系

### 8.1 概念位置

```
IaC：用什么方式描述基础设施（代码化）
GitOps：IaC  + Git 作为唯一真相 + 集群内持续对账
CI/CD：常应用于应用代码；也可用于 IaC 的流水线
```

### 8.2 三种流水线形态

**① 流水线里跑 Terraform（推模型）**

```
PR → CI 跑 terraform plan → 人工审批 → CI 跑 terraform apply
```

- 优点：有审批、有审计。
- 缺点：CI 需要云凭据（风险点）。

**② 流水线里跑 Terraform + 环境隔离**

```
PR → plan（只读凭据）→ 审批 → apply（受限凭据 + OIDC 临时凭证）
```

- 最佳实践：**用 OIDC 联邦获取短期凭据**，不要长期存 access key。

**③ GitOps 管 K8s 层**

```
Git（K8s YAML）→ Argo CD / Flux 在集群内拉取并对账
```

- 凭据只存在集群内。
- 自动检测漂移。

### 8.3 推荐的组合

```
① 基础设施层（VPC/集群）    → Terraform + CI 流水线（plan/apply + 审批）
② K8s 集群内应用           → Argo CD（GitOps）
③ 集群内的平台组件         → 用 Terraform 的 kubernetes/helm provider，或用 Argo CD
④ 策略合规                → Checkov / tfsec / OPA 在 CI 里拦截
```

---

## 九、在 Kubernetes 场景下怎么划边界

这是最容易混的地方：**K8s 里已经有「声明式」了，为什么还要 Terraform？**

### 9.1 边界划分

| 层 | 管什么 | 推荐工具 | 为什么 |
| --- | --- | --- | --- |
| **云基础设施** | VPC、子网、LB、IAM、安全组、托管数据库 | **Terraform** | K8s 管不到这些 |
| **K8s 集群本身** | 控制面、节点池、版本 | **Terraform / Cluster API** | 集群是 K8s 的"基础设施" |
| **集群内平台组件** | CNI、CSI、Ingress、监控 | Helm via Terraform、或 Argo CD | 看团队习惯 |
| **应用** | Deployment/Service | **Argo CD / Flux** | 变更频繁，需要快速对账 |

### 9.2 常见分工方案

**方案 A：Terraform 管到底**

```
Terraform（云资源 + 集群 + 平台组件）→ Argo CD（应用）
```

适合：集群数量少、变更不频繁。

**方案 B：Terraform 管云，集群内全交给 GitOps**

```
Terraform（VPC、集群、节点池）
   ↓ 输出 kubeconfig / cluster 信息
Argo CD（集群内一切：平台组件 + 应用）
```

适合：集群多、平台团队与应用团队分离。

**方案 C：Crossplane（用 K8s 管云）**

```
K8s 里声明云资源 → Crossplane 控制器去创建
```

适合：团队极重度使用 K8s，不想维护两套工具链。缺点是生态与复杂度。

### 9.3 边界不清的典型问题

| 问题 | 后果 |
| --- | --- |
| Terraform 管 K8s 里的 Deployment | 漂移检测冲突、apply 很慢 |
| Argo CD 管云资源 | 需要额外 provision 逻辑，不自然 |
| 两边都管同一个资源 | **互相覆盖**，典型如 Ingress / Service |

**原则**：**同一个对象只能有一个"所有者"**。

---

## 十、实战中的坑

### 10.1 Terraform 常见坑

| 坑 | 表现 | 应对 |
| --- | --- | --- |
| **在控制台手改资源** | 下次 plan 要改回来 | 禁止手动变更，或定期 `plan` 检测漂移 |
| **未加锁并发 apply** | 状态损坏 | 远程后端 + DynamoDB 锁 |
| **`terraform destroy` 误操作** | 生产被毁 | 生产环境禁用 destroy、`prevent_destroy` |
| **状态文件进 Git** | 泄露密钥 | `.gitignore` + 远程后端 |
| **大模块 monolithic** | plan 慢、影响面大 | 按层拆分（network / compute / data） |
| **循环依赖** | plan 报错 | 用模块边界或 `depends_on` 梳理 |
| **provider 版本漂移** | 行为变化 | 锁定版本 `~> 5.0` |
| **import 困难** | 存量资源难纳管 | `terraform import` / `import` 块（1.5+） |

### 10.2 安全与合规

| 措施 | 说明 |
| --- | --- |
| **CI 里做静态扫描** | Checkov、tfsec、Trivy config |
| **策略即代码** | OPA / Sentinel / Conftest 拦截违规（如"禁止公网 S3"） |
| **短时凭据** | OIDC 联邦，不存长期 AK |
| **分环境状态隔离** | prod/dev 各自独立 state 和权限 |
| **变更审批** | plan 输出贴到 PR，人工 review |

### 10.3 组织层面

| 现象 | 问题 | 建议 |
| --- | --- | --- |
| 多个团队各写一套 Terraform | 重复造轮子 | 建模块仓库（内部 registry） |
| 没人看得懂代码 | 黑盒 | 强制 PR review + 文档 |
| 只有一个人会 | 单点 | 轮岗、Pair、文档 |
| 变更没有记录 | 无法追溯 | 全部走 Git + PR |

---

## 十一、面试怎么答

**Q1：什么是 IaC？**

> 用代码描述和管理基础设施。核心价值是可重复、可版本控制、可审查、可重建。它不是简单的"脚本化"，而是把基础设施的**期望状态**声明出来，由工具负责收敛。

**Q2：声明式和命令式的区别？**

> 声明式描述"最终要什么"，天然幂等，可以 diff；命令式描述"一步步做什么"，需要自己处理已存在、失败重试等情况。资源管理用声明式，一次性操作可以用命令式。

**Q3：Terraform 的 State 是干什么的？**

> 它是 Terraform 的"记忆"，记录它管理了哪些资源、ID 是什么、属性如何。`plan` 时它拿代码与 state（刷新后的真实状态）做对比。团队必须用远程后端 + 锁，否则会并发损坏。

**Q4：Terraform 和 Ansible 的区别？**

> Terraform 面向**资源的创建与生命周期**（VPC、VM、LB），声明式；Ansible 面向**服务器上的配置与批量操作**（装包、改配置、跑命令），偏命令式但模块可声明。实践中常配合：Terraform 建机器，Ansible 配置机器。

**Q5：Terraform 和 GitOps 冲突吗？**

> 不冲突，是分工。Terraform 管云资源和集群，Argo CD 管集群内的应用。关键是**同一个资源只能有一个所有者**。

**Q6：如何避免 IaC 改坏生产？**

> plan 必看 + 人工审批、远程状态加锁、生产禁用 destroy、关键资源 `prevent_destroy`、CI 里做静态扫描和策略校验、分环境隔离状态与权限。

**Q7：存量资源怎么纳管？**

> `terraform import`（或 1.5+ 的 `import` 块）把已有资源纳入 state，然后调整代码使其与真实状态一致，`plan` 输出为空才算纳管完成。

**Q8：为什么 Terraform 有些团队换成 OpenTofu？**

> Terraform 从 1.6 起改用 BSL 许可，限制商业竞争使用。OpenTofu 是 Linux 基金会下的开源分支，兼容 HCL 与 provider 协议，适合要求完全开源的场景。

---

## 附：速查表

| 主题 | 一句话 |
| --- | --- |
| IaC 本质 | 用代码声明基础设施的期望状态 |
| 核心价值 | 可重复、可版本化、可审查、可重建 |
| 声明式 vs 命令式 | 要什么 vs 怎么做；前者幂等可 diff |
| Terraform 四元素 | Provider、Resource、Data Source、State |
| State 作用 | 记录"我管了什么"，是实现 plan 的基础 |
| State 存放 | **远程后端 + 锁**，永远不要进 Git |
| 与 Ansible | Terraform 建资源，Ansible 配机器 |
| 与 GitOps | Terraform 管云/集群，Argo CD 管应用 |
| 边界原则 | **一个资源只能有一个所有者** |
| 生产防护 | plan 审批、加锁、禁 destroy、`prevent_destroy`、静态扫描 |
| 安全凭据 | CI 用 OIDC 临时凭证，不存长期 AK |
| 开源替代 | OpenTofu（Terraform 的社区分支） |
