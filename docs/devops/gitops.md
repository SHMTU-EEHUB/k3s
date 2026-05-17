# GitOps / DevOps 目录说明

本仓库将集群声明式配置直接放在仓库根目录下，不再额外包一层 `devops/`。

## 目录职责

- `infrastructure/`：底层基础设施，例如 Longhorn、Mihomo。
- `apps/`：业务与运维应用，例如 Cloudflare DDNS。
- `clusters/`：目标集群聚合入口，例如当前 `master-node`。

## 当前已纳管资源

- `infrastructure/argocd/base`：ArgoCD Helm values 与安装口径。
- `infrastructure/sealed-secrets/base`：Sealed Secrets Helm values 与安装口径。
- `clusters/master-node/cert-manager-app.yaml`：ArgoCD child Application，接管 cert-manager Helm release。
- `clusters/master-node/authentik-app.yaml`：ArgoCD child Application，接管 Authentik Helm release。
- `infrastructure/cert-manager/base`：cert-manager 的 Helm values、Cloudflare DNS-01 所需的加密 Secret，以及 Let’s Encrypt ClusterIssuer（由 root Kustomize / ArgoCD 同步）。
- `infrastructure/authentik/base`：Authentik 的命名空间、Helm values、加密后的 `authentik-config`（由 root Kustomize / ArgoCD 同步）。
- `infrastructure/mihomo/base`：Mihomo 网关、MetaCubeXD UI、控制器 / Web UI NodePort、ClusterIP / Headless Service、加密后的 `mihomo-config`。
- `infrastructure/longhorn/base`：Longhorn StorageClass、单节点副本设置、`master1` 磁盘声明，以及系统 SSD 先纳入 `fast` 层、后迁 worker SSD 的策略。

- `apps/cloudflare-ddns/base`：Cloudflare DDNS ConfigMap、Deployment、加密后的 `cloudflare-ddns-secret`。
- `apps/ai-services/base`：AI 服务组，包括共享 PostgreSQL（`longhorn-fast-1replica`）、Metapi、Aether（Rust Pioneer）+ 专用 Redis、GPT-Load 与 Codex2API 共用 Redis、OutlookEmail、Kiro、CLIProxyAPI、Grok2API，以及加密后的 `secret-sealed.yaml`。
- `clusters/master-node`：当前单 Master 集群聚合入口。

## 敏感信息原则

以下内容不能提交明文：

- Cloudflare API Token。
- cert-manager / ACME 用于 DNS-01 的 Cloudflare API Token。
- Authentik `secret_key`、数据库口令、SMTP 口令、OIDC / OAuth Client Secret。
- Mihomo 订阅链接、代理节点密码、UUID、私钥、控制器密钥。
- Metapi、Aether、Kiro、CLIProxyAPI、Grok2API、GPT-Load、Codex2API、OutlookEmail 的 API Key、管理 Token、数据库口令、Redis 口令、登录口令、`SECRET_KEY` 与 OAuth / Refresh Token。
- Kubeconfig、K3s token、Tailscale Auth Key、Tailnet ACL 私有策略。

当前仓库默认使用 Sealed Secrets 管理敏感信息。明文 Secret 只允许作为本地临时文件存在，生成 `SealedSecret` 后必须删除，不能提交。对 cert-manager 和 Authentik 这类 Helm 应用，不再手工 `helm upgrade`，统一交给 ArgoCD child Application 接管。
