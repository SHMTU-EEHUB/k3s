# master-node

当前单 Master 集群的最终状态聚合目录。

## 包含内容

- `../../infrastructure/longhorn/base`
- `../../infrastructure/cert-manager/base`
- `../../infrastructure/authentik/base`
- `../../infrastructure/mihomo/base`
- `../../apps/cloudflare-ddns/base`
- `../../apps/ai-services/base`

Sealed Secrets Controller 由 Helm 管理，配置见 `../../infrastructure/sealed-secrets/base`。Controller 必须先于本目录中的 SealedSecret 资源安装。

## 使用方式

本目录可用作手工 Kustomize 应用入口，也可作为 ArgoCD Application 的 `path`。

注意：`sync-app.yaml` 是 ArgoCD 的 Application 示例，不放入 `kustomization.yaml`。如果集群尚未安装 ArgoCD CRD，不要直接 `kubectl apply` 该文件。

## 上线前检查

- Sealed Secrets Controller 已通过 Helm 安装。
- cert-manager Helm Release 已通过 `../../infrastructure/cert-manager/base/values.yaml` 安装，且 CRD 已就绪。
- `cloudflare-api-token` 由 `infrastructure/cert-manager/base/secret-sealed.yaml` 解封生成。
- `authentik-config` 由 `infrastructure/authentik/base/secret-sealed.yaml` 解封生成。
- `cloudflare-ddns-secret` 由 `apps/cloudflare-ddns/base/secret-sealed.yaml` 解封生成。
- `mihomo-config` 由 `infrastructure/mihomo/base/secret-sealed.yaml` 解封生成。
- `ai-postgresql-secret`、`metapi-secret`、`aether-secret`、`ds2api-secret`、`kiro-rs-secret`、`cli-proxy-api-secret`、`grok2api-secret` 由 `apps/ai-services/base/secret-sealed.yaml` 解封生成。
- Longhorn CRD 已由 Helm 安装完成。

cert-manager Helm Release 建议使用 `../../infrastructure/cert-manager/base/values.yaml` 手工执行 `helm upgrade --install`，待其 CRD 就绪后再同步本目录中的 `ClusterIssuer`。

Authentik Helm Release 建议使用 `../../infrastructure/authentik/base/values.yaml` 手工执行 `helm upgrade --install`，与当前 ArgoCD / Sealed Secrets 的 DevOps 口径保持一致。
