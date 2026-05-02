# master-node

当前单 Master 集群的最终状态聚合目录。

## 包含内容

- `../../infrastructure/longhorn/base`
- `../../infrastructure/mihomo/base`
- `../../apps/cloudflare-ddns/base`

## 使用方式

本目录可用作手工 Kustomize 应用入口，也可作为 ArgoCD Application 的 `path`。

注意：`sync-app.yaml` 是 ArgoCD 的 Application 示例，不放入 `kustomization.yaml`。如果集群尚未安装 ArgoCD CRD，不要直接 `kubectl apply` 该文件。

## 上线前检查

- `cloudflare-ddns-secret` 已存在，或已替换为真实 SealedSecret。
- `cloudflare-ddns-config` 中的 Zone ID 已通过私有 overlay 或 Secret 处理。
- `clash-config` 已替换为完整私有 Mihomo 配置。
- Longhorn CRD 已由 Helm 安装完成。
