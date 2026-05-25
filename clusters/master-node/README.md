# master-node

当前单 Master 集群的最终状态聚合目录。

## 包含内容

- `./cert-manager-app.yaml`
- `./authentik-app.yaml`
- `../../infrastructure/longhorn/base`
- `../../infrastructure/cert-manager/base`
- `../../infrastructure/authentik/base`
- `../../infrastructure/mihomo/base`
- `../../apps/cloudflare-ddns/base`
- `../../apps/ai-services/base`
- `../../apps/inventree/base`

Sealed Secrets Controller 由 Helm 管理，配置见 `../../infrastructure/sealed-secrets/base`。Controller 必须先于本目录中的 SealedSecret 资源安装。

## 使用方式

本目录可用作手工 Kustomize 应用入口，也可作为 ArgoCD Application 的 `path`。

注意：`sync-app.yaml` 是 ArgoCD 的 Application 示例，不放入 `kustomization.yaml`。如果集群尚未安装 ArgoCD CRD，不要直接 `kubectl apply` 该文件。

## 上线前检查

- Sealed Secrets Controller 已通过 Helm 安装。
- cert-manager child Application 已由 ArgoCD 同步，且 CRD 已就绪。
- `cloudflare-api-token` 由 `infrastructure/cert-manager/base/secret-sealed.yaml` 解封生成。
- `authentik-config` 由 `infrastructure/authentik/base/secret-sealed.yaml` 解封生成。
- `cloudflare-ddns-secret` 由 `apps/cloudflare-ddns/base/secret-sealed.yaml` 解封生成。
- `mihomo-config` 由 `infrastructure/mihomo/base/secret-sealed.yaml` 解封生成。
- `ai-postgresql-secret`、`metapi-secret`、`aether-secret`、`ai-services-redis-secret`、`kiro-rs-secret`、`cli-proxy-api-secret`、`grok2api-secret`、`gpt-load-secret`、`codex2api-secret`、`halowebui-secret`、`outlook-email-secret` 由 `apps/ai-services/base/secret-sealed.yaml` 解封生成。
- `inventree-postgresql-secret`、`inventree-redis-secret`、`inventree-secret` 由 `apps/inventree/base/secret-sealed.yaml` 解封生成。
- Longhorn CRD 已由 Helm 安装完成。

- cert-manager Helm Release 由 `./cert-manager-app.yaml` 接管，`ClusterIssuer` 仍由本目录中的 Kustomize 资源管理。
- Authentik Helm Release 由 `./authentik-app.yaml` 接管，`authentik-config` 仍由本目录中的 SealedSecret 管理。
