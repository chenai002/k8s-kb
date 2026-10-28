# Terraform 详解：基本概念与处理流程

> 从「它是什么」到「一条命令背后发生了什么」，把 Terraform 的核心概念和完整流程讲透。
> 前置阅读：[`iac.md`](./iac.md)（IaC 的整体背景与工具选型）

---

## 目录

- [一、Terraform 是什么](#一terraform-是什么)
- [二、核心概念](#二核心概念)
- [三、配置文件结构](#三配置文件结构)
- [四、处理流程：一条命令背后发生了什么](#四处理流程一条命令背后发生了什么)
- [五、State 深入](#五state-深入)
- [六、依赖与执行顺序](#六依赖与执行顺序)
- [七、多环境组织方式](#七多环境组织方式)
- [八、团队协作与 CI/CD](#八团队协作与-cicd)
- [九、常见坑与最佳实践](#九常见坑与最佳实践)
- [十、面试怎么答](#十面试怎么答)
- [附：命令速查](#附命令速查)

---

## 一、Terraform 是什么

> **一个声明式的基础设施编排工具**：你写「我要什么」，它负责算出「怎么变过去」，并调用各家云厂商的 API 去执行。

```
你写的 HCL 配置  ──▶  Terraform  ──▶  云厂商 API（AWS/GCP/Azure/K8s/...）
   （期望状态）         （对比+规划）        （真实资源）
```

**三个特征**：

| 特征 | 说明 |
| --- | --- |
| **声明式** | 描述结果，不写步骤；天然幂等 |
| **多云** | 通过 Provider 插件对接上千种 API |
| **有状态** | 用 state 文件记住「自己管了哪些资源」 |

**它不是什么**：
- 不是配置管理工具（那是 Ansible 的活，虽然它也能装软件）
- 不是运行时平台（不负责应用部署后的运行）
- 不是 IaC 的唯一选择（Pulumi/CloudFormation 也能做）

---

## 二、核心概念

Terraform 的概念可以归成 **五类**：配置侧（写什么）、执行侧（怎么变）、状态侧（记住什么）。

### 2.1 Provider（提供者）

**与某个 API 打交道的插件。**

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"      # 锁定大版本
    }
  }
}

provider "aws" {
  region = "ap-east-1"
}
```

- Provider 由 HashiCorp 或社区维护，一个 Provider 提供几十到几百个资源类型。
- `terraform init` 会下载 Provider 二进制到 `.terraform/` 目录。
- **版本要锁**：否则 `init` 可能拉到不兼容的新版本。

### 2.2 Resource（资源）

**一个被管理的对象。** 是 Terraform 的核心单位。

```hcl
resource "aws_instance" "web" {        # 类型          名字
  ami           = "ami-12345678"
  instance_type = "t3.micro"
}
```

- **类型**（`aws_instance`）由 Provider 定义。
- **名字**（`web`）是你在配置里给它起的标识，用来在模块内引用。
- 资源在状态里的地址是 `aws_instance.web`。

**生命周期控制**：

```hcl
resource "aws_instance" "web" {
  # ...
  lifecycle {
    prevent_destroy      = true     # 禁止被 destroy
    create_before_destroy = true     # 先建新的，再删旧的（减少中断）
    ignore_changes       = [tags]    # 外部改动的字段不追踪
  }
}
```

### 2.3 Data Source（数据源）

**读取已有信息，但不管理它。**用 `data` 关键字声明。

```hcl
# 查最新的 Ubuntu AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

resource "aws_instance" "web" {
  ami = data.aws_ami.ubuntu.id      # 引用数据源
}
```

**Resource vs Data Source**：

| | Resource | Data Source |
| --- | --- | --- |
| 是否管理生命周期 | ✅ 会创建/修改/删除 | ❌ 只读 |
| 是否进 state | ✅ 存完整属性 | ✅ 存读取结果（每次刷新） |
| 用途 | 我要的东西 | 已存在的东西 |

### 2.4 Variable / Output / Local

| 概念 | 作用 | 例子 |
| --- | --- | --- |
| **variable** | 输入参数 | `variable "env" {}` |
| **output** | 输出结果（模块间传递、给 CI 用） | `output "vpc_id" {}` |
| **locals** | 中间变量（不暴露、不接收） | `locals { name = "${var.env}-app" }` |

```hcl
variable "env" {
  type        = string
  default     = "dev"
  description = "环境名"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.env)
    error_message = "env 只能是 dev/staging/prod"
  }
}

locals {
  name_prefix = "${var.env}-shop"
  common_tags = {
    Environment = var.env
    ManagedBy   = "terraform"
  }
}

output "bucket_arn" {
  value = aws_s3_bucket.logs.arn
}
```

**变量来源优先级**（高 → 低）：

```
命令行 -var / -var-file
  > *.auto.tfvars
  > terraform.tfvars
  > 环境变量 TF_VAR_xxx
  > default
```

### 2.5 Module（模块）

**一组资源的封装，可复用、可分层。**

```hcl
module "network" {
  source = "./modules/network"        # 本地 / Git / Registry
  cidr   = "10.0.0.0/16"
  azs    = ["ap-east-1a", "ap-east-1b"]
}

resource "aws_instance" "app" {
  subnet_id = module.network.private_subnet_ids[0]   # 用模块输出
}
```

- 模块 = 一个包含 `.tf` 文件的目录，有 `variables.tf`（输入）和 `outputs.tf`（输出）。
- 来源可以是：本地路径、Git 仓库（含 `?ref=v1.2.0` 锁版本）、Terraform Registry。
- **根模块** vs **子模块**：你运行 `terraform apply` 的目录是根模块。

### 2.6 State（状态）

**Terraform 的"记忆"，记录它管理了什么。** 单独一章讲（见第五节）。

---

## 三、配置文件结构

一个典型项目的目录：

```
infra/
├── main.tf            # 主要资源
├── variables.tf       # 输入变量
├── outputs.tf         # 输出
├── versions.tf        # terraform 版本与 provider 版本约束
├── providers.tf       # provider 配置
├── backend.tf         # 远程状态后端
├── terraform.tfvars   # 变量值（环境相关，可不入 Git）
└── modules/
    ├── network/
    └── compute/
```

### 3.1 最小可运行示例

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
  backend "s3" {
    bucket         = "my-tfstate"
    key            = "dev/network.tfstate"
    region         = "ap-east-1"
    encrypt        = true
    dynamodb_table = "tf-lock"
  }
}

# main.tf
provider "aws" { region = "ap-east-1" }

resource "aws_s3_bucket" "logs" {
  bucket = "${var.env}-logs"
  tags   = local.common_tags
}

# variables.tf
variable "env" {
  type    = string
  default = "dev"
}

# locals.tf
locals {
  common_tags = { Env = var.env, ManagedBy = "terraform" }
}

# outputs.tf
output "bucket_name" { value = aws_s3_bucket.logs.bucket }
```

---

## 四、处理流程：一条命令背后发生了什么

这是 understanding Terraform 最关键的部分。

### 4.1 总览

```
terraform init   → 准备环境（下载 provider / modules、初始化 backend）
terraform plan   → 计算差异，生成执行计划（只读，不改资源）
terraform apply  → 执行计划（写资源、更新 state）
terraform destroy→ 销毁资源
```

### 4.2 `terraform init` 做了什么

```
① 读取配置，解析 required_providers
② 下载 Provider 插件到 .terraform/providers/
③ 下载引用的 modules 到 .terraform/modules/
④ 初始化 backend（连接远程 state 存储）
⑤ 生成/更新 .terraform.lock.hcl（锁定 provider 版本与校验和）
```

**可重复执行**，无副作用（除非 backend 配置变了需要 `-migrate-state`）。

### 4.3 `terraform plan` 的内部步骤（重点）

```
① 配置解析
   读取所有 .tf 文件，构建出"期望的资源图"

② 状态读取
   从 backend 加载 state 文件，得到"上次我管理了什么"

③ 刷新（refresh，可跳过）
   对 state 里每个资源调用云 API 的 Read 操作，
   获取"现在的真实属性"
   ⚠️ 这一步会消耗 API 配额，大项目要注意

④ 构建依赖图（DAG）
   根据资源间的引用关系，构建有向无环图，
   决定创建/更新/删除的顺序

⑤ Diff 计算
   对比"期望"与"真实"，逐资源判断：
     · 不存在 → create
     · 存在且属性不同 → update (in-place) 或 replace
     · 存在于 state 但配置里没有 → delete

⑥ 输出计划
   显示 +0 ~1 -0 之类的摘要，以及每个资源的具体变更
```

**符号含义**：

| 符号 | 含义 |
| --- | --- |
| `+` | 新建 |
| `~` | 原地更新（in-place） |
| `-` | 删除 |
| `-/+` | **先删后建**（force replacement） |
| `+/-` | **先建后删**（create_before_destroy） |
| `<=` | 只读数据源 |

**重要区分：update vs replace**

```
update  ：改一个可原地修改的属性（如 tag、安全组规则）→ 资源不重建
replace ：改了不可变的属性（如实例的 AMI、子网 ID）→ 必须销毁重建
```

**replace 是高危操作**：对有状态资源（数据库）意味着可能丢数据。`plan` 里出现 `# forces replacement` 一定要仔细看。

### 4.4 `terraform apply` 做了什么

```
① 再跑一次 plan（除非 -refresh=false 或用了保存的 plan 文件）
② 按依赖图顺序执行变更
③ 每完成一个资源，就写入一次 state（不是最后统一写）
④ 失败时：
   · 已完成的资源写入 state
   · 继续还是中止取决于依赖关系
   · 可以再次 apply 继续（幂等）
```

**推荐做法：用保存的计划文件**

```bash
terraform plan -out=tfplan      # 生成计划并保存
terraform apply tfplan          # 严格按这个计划执行
```

**好处**：避免「plan 时看到的」和「apply 时执行的」不一致（中间可能有人改了配置或云上资源）。

### 4.5 `terraform destroy`

```
① 反向遍历依赖图（先删最外层）
② 逐个调用删除 API
③ 从 state 移除
```

**防护**：

```hcl
lifecycle { prevent_destroy = true }   # 关键资源加锁
```

---

## 五、State 深入

### 5.1 三方关系

```
         ┌──────────────┐
你写的配置 │  期望状态     │
           └──────┬───────┘
                  │ ② diff
           ┌──────▼───────┐
状态文件    │  State       │  ← 记录"我管了哪些资源、ID、属性、依赖"
           └──────┬───────┘
                  │ ① refresh
           ┌──────▼───────┐
云上        │  真实资源     │
           └──────────────┘
```

**为什么需要 state**：
1. 云上资源没有"名字"，只有 ID。Terraform 必须记住「`aws_instance.web` 对应 `i-0abc123`」。
2. 判断该创建还是更新，靠的是 state 里的记录。
3. 删除时要知道删哪些。

### 5.2 状态的位置

| 方式 | 场景 | 说明 |
| --- | --- | --- |
| 本地 `terraform.tfstate` | 学习、个人 | **绝对不能提交 Git** |
| **S3 + DynamoDB** | AWS 团队常用 | DynamoDB 提供状态锁 |
| **GCS / Azure Blob** | 其他云 | 各自有锁机制（或使用服务端锁） |
| **Terraform Cloud / OpenTofu** | 团队 | 托管状态 + 运行 + 审批 |

### 5.3 状态锁

```
A 执行 apply ──▶ 获取锁
B 也执行 apply ──▶ 拿不到锁 → 报错 "Error acquiring the state lock"
```

**没有锁的后果**：两人同时 apply，state 互相覆盖，导致 Terraform 以为自己管的资源和实际不符 → 可能误删。

**手动解锁**（确认没有其他进程在跑）：

```bash
terraform force-unlock <LOCK_ID>
```

### 5.4 状态操作命令

| 命令 | 用途 |
| --- | --- |
| `terraform state list` | 列出 state 里的所有资源 |
| `terraform state show <addr>` | 查看某个资源的详细属性 |
| `terraform state mv A B` | 重命名/移动资源（重构时用） |
| `terraform state rm <addr>` | **从 state 移除**（不再管理，但不删真实资源） |
| `terraform import <addr> <id>` | 把已有资源纳管进 state |
| `terraform refresh` | 刷新 state（现在推荐 `plan -refresh-only`） |

**`state rm` 与 `destroy` 的区别**：

```
state rm  → Terraform 忘了它，但资源还在云上
destroy   → 真的去删资源
```

### 5.5 常见状态问题

| 问题 | 现象 | 解决 |
| --- | --- | --- |
| 资源被手动删了 | plan 显示要重建 | 接受重建，或 `state rm` 后重新 import |
| 资源被手动改了 | plan 显示要改回去 | `ignore_changes` 或禁止手改 |
| 想重命名资源 | 会变成"删了再建" | 用 `state mv` 改地址 |
| 状态损坏 | 各种奇怪报错 | 从备份恢复（远程后端通常有版本） |

---

## 六、依赖与执行顺序

### 6.1 隐式依赖（推荐）

**通过引用建立依赖**，Terraform 自动排序：

```hcl
resource "aws_vpc" "main" { cidr_block = "10.0.0.0/16" }

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id        # ← 隐式依赖：先建 VPC
  cidr_block = "10.0.1.0/24"
}
```

### 6.2 显式依赖

**配置里看不出引用关系时**，用 `depends_on`：

```hcl
resource "aws_instance" "app" {
  # ...
  depends_on = [aws_iam_role_policy_attachment.app]
}
```

**能用隐式依赖就别用 `depends_on`**——后者会让计划变得不精确。

### 6.3 依赖图（DAG）

```
      aws_vpc.main
           │
     ┌─────┴─────┐
aws_subnet.a  aws_subnet.b
     │             │
     └─────┬───────┘
     aws_instance.app
```

- **创建**：从根往下
- **删除**：从叶往上
- **并行**：同一层的资源会**并行**操作（`-parallelism` 控制，默认 10）

---

## 七、多环境组织方式

### 7.1 方式一：目录隔离（最常用、最安全）

```
infra/
├── environments/
│   ├── dev/
│   │   ├── main.tf        # 每个环境独立 state
│   │   └── backend.tf     # 指向 dev 的 state key
│   ├── staging/
│   └── prod/
└── modules/
    └── network/
```

- **优点**：状态完全隔离，误操作不会波及别的环境。
- **缺点**：改动要复制到多个环境（用模块 + 少量差异缓解）。

### 7.2 方式二：Workspace

```bash
terraform workspace new prod
terraform workspace select prod
terraform apply
```

- 同一个目录，state 按 workspace 分开（`env:/prod/xxx.tfstate`）。
- **缺点**：容易选错 workspace 导致误操作；`terraform destroy` 在错误的 workspace 上很危险。
- **建议**：workspace 适合**临时/测试**环境，生产用目录隔离。

### 7.3 方式三：Terragrunt

在 Terraform 之上加一层封装，自动生成 backend 与 provider 配置，减少重复：

```hcl
# terragrunt.hcl
terraform { source = "../../modules/network" }
include "root" { path = find_in_parent_folders() }
inputs = { cidr = "10.0.0.0/16" }
```

适合**大量环境 × 大量模块**的团队。

---

## 八、团队协作与 CI/CD

### 8.1 典型流水线

```
PR 提交
  │
  ├─ terraform fmt -check        # 格式
  ├─ terraform validate          # 语法
  ├─ tflint / checkov / tfsec    # 静态扫描
  ├─ terraform plan              # 生成计划
  └─ 把 plan 输出贴到 PR 评论      # 人工 review
       │
       ▼ 合并且审批通过
  terraform apply tfplan         # 用保存的计划执行
```

### 8.2 关键实践

| 实践 | 原因 |
| --- | --- |
| **plan 结果贴到 PR** | 变更可见、可审查 |
| **保存 plan 文件再 apply** | 保证"看到的就是执行的" |
| **apply 用审批门禁** | 生产变更需人确认 |
| **OIDC 临时凭据** | 不在 CI 里存长期 AK |
| **分环境独立 state 与权限** | 生产 state 只允许特定流水线访问 |
| **禁止在本地直接 apply 生产** | 避免绕过审计 |
| **定期 `plan -refresh-only` 检测漂移** | 发现手动改动 |

### 8.3 与 GitOps 的关系

| 层 | 推荐 |
| --- | --- |
| 云资源 / K8s 集群 | **Terraform + CI（plan/apply + 审批）** |
| K8s 集群内的应用 | **Argo CD / Flux（GitOps）** |

**两者的边界**：同一资源只能有一个所有者。

---

## 九、常见坑与最佳实践

### 9.1 坑

| 坑 | 表现 | 应对 |
| --- | --- | --- |
| **state 进 Git** | 泄露密钥 | `.gitignore` + 远程后端 |
| **没开状态锁** | 并发 apply 损坏 state | 远程后端 + 锁 |
| **手动改云上资源** | 下次 plan 要改回来 | 禁止手改，或 `ignore_changes` |
| **`-/+` 没注意** | 数据库被重建 | plan 必看 `forces replacement` |
| **provider 版本漂移** | 行为不一致 | 提交 `.terraform.lock.hcl` |
| **一个大 state** | plan 慢、影响面大 | 按层/按环境拆分 |
| **`count` 加中间元素** | 索引错位导致重建 | 用 `for_each` + map 代替 `count` |
| **`terraform destroy` 打错环境** | 生产被毁 | 生产禁 destroy、`prevent_destroy` |
| **敏感值出现在 plan 输出** | 泄漏 | `sensitive = true`、用 Vault/SSM 引用 |

### 9.2 `count` vs `for_each`（重要）

```hcl
# ❌ 用 count：中间删一个，后面的索引全变 → 资源被重建
resource "aws_instance" "web" {
  count = 3
}

# ✅ 用 for_each：按 key 管理，增删不影响其他
resource "aws_instance" "web" {
  for_each = toset(["a", "b", "c"])
}
```

### 9.3 最佳实践清单

- ✅ 所有配置文件进 Git，包括 `.terraform.lock.hcl`
- ✅ state 用远程后端 + 加密 + 锁
- ✅ 模块化：网络 / 计算 / 数据分层
- ✅ 变量加 `type`、`description`、必要的 `validation`
- ✅ 资源加统一 `tags`（Env、Owner、ManagedBy）
- ✅ provider 与 terraform 版本都锁
- ✅ 生产关键资源加 `prevent_destroy`
- ✅ CI 里跑 `fmt / validate / tflint / plan`
- ✅ 用 `for_each` 而非 `count` 管理异构资源
- ✅ plan 必看，尤其是 `forces replacement`

---

## 十、面试怎么答

**Q1：Terraform 的执行流程？**

> `init` 下载 provider 和 modules、初始化后端；`plan` 读取配置和 state，刷新真实状态，构建依赖图并计算 diff，生成计划；`apply` 按计划执行并逐步写回 state。生产上推荐 `plan -out` 保存计划再 `apply`，保证看到的就是执行的。

**Q2：State 为什么必要？**

> 云资源只有 ID 没有名字。state 记录「配置里的逻辑名 ↔ 云上资源 ID」的映射，同时保存属性和依赖关系。没有 state 就无法判断资源是新建还是更新，也无法知道该删什么。

**Q3：update 和 replace 的区别？**

> update 是原地修改（如改 tag）；replace 是销毁重建（如改实例 AMI）。plan 里出现 `forces replacement` 就是后者，对有状态资源是高危信号。

**Q4：如何避免并发修改 state？**

> 使用支持锁的远程后端（如 S3 + DynamoDB、Terraform Cloud），apply 时自动加锁；异常退出后确认无进程再 `force-unlock`。

**Q5：存量资源怎么纳管？**

> 用 `terraform import`（或 1.5+ 的 `import` 块）把资源加入 state，再调整配置使其与真实属性一致，直到 `plan` 输出为空。

**Q6：多环境怎么组织？**

> 推荐**目录隔离**（每个环境独立 state 与 backend）；workspace 更轻便但容易选错环境，适合临时环境；环境特别多时可用 Terragrunt 减少重复。

**Q7：Terraform 和 ARM/CloudFormation 有什么区别？**

> 都是声明式 IaC，但 Terraform 多云、Provider 生态广、有独立的 state 与 plan 机制；CloudFormation 是 AWS 原生托管，无需自己管状态，但绑定 AWS。

---

## 附：命令速查

| 命令 | 用途 |
| --- | --- |
| `terraform init` | 初始化（下载 provider/module、连后端） |
| `terraform init -upgrade` | 升级 provider 到允许的最新版本 |
| `terraform plan` | 生成执行计划 |
| `terraform plan -out=tfplan` | 保存计划到文件 |
| `terraform plan -target=aws_instance.web` | 只对某个资源做计划 |
| `terraform apply` | 执行计划 |
| `terraform apply tfplan` | 执行指定计划文件 |
| `terraform destroy` | 销毁所有管理中的资源 |
| `terraform fmt -recursive` | 格式化代码 |
| `terraform validate` | 语法与配置校验 |
| `terraform output` | 查看输出值 |
| `terraform state list` | 列出 state 中的资源 |
| `terraform state show <addr>` | 查看某资源属性 |
| `terraform state mv <src> <dst>` | 重命名/移动资源 |
| `terraform state rm <addr>` | 从 state 移除（不删真实资源） |
| `terraform import <addr> <id>` | 导入已有资源 |
| `terraform force-unlock <id>` | 强制解锁（确认无进程后） |
| `terraform providers` | 列出使用的 provider |
| `terraform graph` | 输出依赖图（DOT 格式） |
| `terraform console` | 交互式测试表达式 |
