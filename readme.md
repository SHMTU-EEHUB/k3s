# k3s-eehub

> 轻量 DevOps / K3s 基础设施文档仓库。

## 文档入口

- `docs/machines.md`：硬件配置架构与当前集群状态。
- `docs/init.md`：从 0 安装 / 恢复集群的流程与验收。
- `docs/basement.md`：Longhorn 与 Mihomo 的基础设施介绍。
- `docs/devops/gitops.md`：当前 GitOps / DevOps 目录说明。

## DevOps / GitOps 入口

- `infrastructure/`：底层基础设施，例如 Longhorn、Mihomo。
- `apps/`：业务与运维应用，例如 Cloudflare DDNS。
- `clusters/master-node/`：当前 `master1` 单节点阶段的集群聚合入口。

历史聊天记录和旧 K8s 方案保存在 `archive/` 下，仅用于追溯。
