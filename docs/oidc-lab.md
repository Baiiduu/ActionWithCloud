# GitHub Actions OIDC 实验说明

这个实验用于研究 GitHub Actions 如何通过 OIDC 把 workflow/job 的身份传播到云平台。当前阶段只做最小、只读、手动触发的身份验证，不做部署、不写入云资源、不推送制品。

## 当前目标

第一步不是攻击云平台，而是看清楚这条身份链：

```text
GitHub workflow/job
  -> permissions: id-token: write
  -> GitHub 颁发 OIDC token
  -> 云平台验证 token claims
  -> 云平台授予临时身份
  -> workflow 获得云侧访问能力
```

你现在已经完成了第一小步：成功获得并解码了 GitHub OIDC token 的 claims。

## 已添加的 Workflows

- `.github/workflows/oidc-claims.yml`
  - 用途：查看 GitHub OIDC token 中的关键 claims。
  - 只打印 claims，不打印原始 token。

- `.github/workflows/aws-oidc.yml`
  - 用途：让 GitHub Actions 通过 OIDC 换取 AWS 临时身份。
  - 验证命令：`aws sts get-caller-identity`。

- `.github/workflows/azure-oidc.yml`
  - 用途：让 GitHub Actions 通过 OIDC 登录 Azure。
  - 验证命令：`az account show`。

- `.github/workflows/gcp-oidc.yml`
  - 用途：让 GitHub Actions 通过 OIDC 登录 Google Cloud。
  - 验证命令：`gcloud auth list`。

## AWS / Azure / GCP 是什么

AWS、Azure、GCP 是三个主流云平台：

- AWS：Amazon Web Services，亚马逊云。
  - 典型资源：IAM Role、EC2、S3、ECR、Lambda、CloudFormation。
  - 在本实验中，GitHub OIDC token 会被 AWS STS 换成一个 IAM Role 的临时凭证。

- Azure：Microsoft Azure，微软云。
  - 典型资源：Subscription、Resource Group、App Service、AKS、Azure Container Registry。
  - 在本实验中，GitHub OIDC token 会被 Azure federated credential 机制换成一个 Azure 应用身份的登录状态。

- GCP：Google Cloud Platform，也叫 Google Cloud，谷歌云。
  - 典型资源：Service Account、GKE、Cloud Run、Artifact Registry、Cloud Storage。
  - 在本实验中，GitHub OIDC token 会通过 Workload Identity Federation 换成一个 GCP Service Account 的身份。

从论文角度看，这三个平台都是“云侧信任边界”。GitHub Actions 本来只是在 GitHub 里运行，但通过 OIDC，它可以被云平台信任，进而获得云侧权限。

## GitHub Repository Variables

下面这些值应该配置为 GitHub 仓库变量，而不是 secrets。因为它们是身份标识符，不是私密密码。

AWS 需要：

- `AWS_ROLE_ARN`
- `AWS_REGION`

Azure 需要：

- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID`

GCP 需要：

- `GCP_WORKLOAD_IDENTITY_PROVIDER`
- `GCP_SERVICE_ACCOUNT`

## 你已经观察到的 Claims

这次运行中，最关键的字段是：

```text
sub = repo:Baiiduu/ActionWithCloud:ref:refs/heads/main
aud = oidc-lab
repository = Baiiduu/ActionWithCloud
ref = refs/heads/main
workflow = OIDC Claims Inspector
event_name = workflow_dispatch
```

其中 `sub` 是后续云平台 trust policy 最常用的判断字段。

它的含义是：

```text
这个 token 来自 Baiiduu/ActionWithCloud 仓库的 main 分支
```

如果后面 AWS、Azure 或 GCP 配置信任这个 `sub`，那么这个 GitHub workflow 就可以换取云平台身份。

## 最小云侧信任原则

刚开始实验时，云侧 trust policy 应该尽量窄：

- 只信任当前仓库：`Baiiduu/ActionWithCloud`。
- 只信任当前分支：`refs/heads/main`。
- 只授予只读权限。
- 不要一开始给部署、删除、推送镜像、修改 IAM 等高危权限。

后续研究时，可以逐步构造对照组：

- 只允许 `main` 分支。
- 允许整个仓库。
- 允许整个 owner。
- 只检查 `aud`，不检查 `sub`。
- 允许第三方 reusable workflow 或 third-party action 参与拥有 OIDC 权限的 job。

这些对照组可以帮助我们测量：哪些配置会让 GitHub 侧的 workflow 信任扩张成云侧权限风险。

## 后续研究记录

每次实验建议记录：

- GitHub workflow 文件名。
- 触发方式，例如 `workflow_dispatch`、`push`、`pull_request`。
- `permissions` 配置。
- `sub` 字段。
- `aud` 字段。
- 云平台 trust policy。
- 云侧授予的身份。
- 云侧实际权限。
- 是否使用第三方 action。
- 是否使用 mutable ref，例如 `@main`、`@master`。

这些记录最终可以转化成论文里的 identity propagation graph 和 high-risk chain 检测规则。
