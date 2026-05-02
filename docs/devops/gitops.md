# GitOps / DevOps 目录说明

本仓库将集群声明式配置直接放在仓库根目录下，不再额外包一层 `devops/`。

## 目录职责

- `infrastructure/`：底层基础设施，例如 Longhorn、Mihomo。
- `apps/`：业务与运维应用，例如 Cloudflare DDNS。
- `clusters/`：目标集群聚合入口，例如当前 `master-node`。

## 当前已纳管资源

- `infrastructure/mihomo/base`：Mihomo 网关、MetaCubeXD UI、NodePort Service、Headless Service。
- `infrastructure/longhorn/base`：Longhorn StorageClass、单节点副本设置、`master1` 磁盘声明。
- `apps/cloudflare-ddns/base`：Cloudflare DDNS ConfigMap 与 Deployment。
- `clusters/master-node`：当前单 Master 集群聚合入口。

## 敏感信息原则

以下内容不能提交明文：

- Cloudflare API Token。
- Mihomo 订阅链接、代理节点密码、UUID、私钥、控制器密钥。
- Kubeconfig、K3s token、Tailscale Auth Key。

推荐使用 Sealed Secrets、External Secrets 或私有 overlay 管理。
