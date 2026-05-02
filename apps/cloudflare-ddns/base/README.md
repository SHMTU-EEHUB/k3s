# Cloudflare DDNS

当前对应 `master1` 上的 `cloudflare-ddns` Deployment。

## 敏感信息

`CF_API_TOKEN` 不提交到仓库。请用以下任一方式在集群中准备 `cloudflare-ddns-secret`：

- 手动创建 Kubernetes Secret。
- 使用 Sealed Secrets，将加密后的结果替换到 `secret-sealed.yaml`，并在 `kustomization.yaml` 中显式加入该文件。

`ZONE_ID` 在当前仓库中使用占位符，若你希望完全 GitOps 化，也应放入私有 overlay 或 Secret 管理。

## 当前运行口径

- 命名空间：`default`
- Workload：`Deployment/cloudflare-ddns`
- 镜像：`favonia/cloudflare-ddns:latest`
- 副本数：`1`
- 调度：固定到控制平面节点
- 更新周期：`@every 5m`
