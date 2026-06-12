# AGENTS

## 项目结构

| 路径                 | 说明                                                                      |
| -------------------- | ------------------------------------------------------------------------- |
| `AGENTS.md`          | AI Agent 上下文文件，保留在仓库根目录                                     |
| `readme.md`          | 仓库入口说明                                                              |
| `docs/machines.md`   | 硬件配置架构：机器、磁盘、网络、K3s、Longhorn、Tailscale、Mihomo 当前状态 |
| `docs/init.md`       | 从 0 安装 / 恢复集群的步骤、路线和验收内容                                |
| `docs/basement.md`   | 基础设施介绍：Longhorn、Tailscale、Mihomo                                 |
| `docs/devops/`       | DevOps / GitOps 目录说明与操作原则                                        |
| `infrastructure/`    | 底层基础设施声明式配置：Sealed Secrets、Longhorn、Tailscale、Mihomo 等    |
| `apps/`              | 应用层声明式配置：Cloudflare DDNS 等                                      |
| `clusters/`          | 集群最终状态聚合入口                                                      |
| `archive/chat-logs/` | 历史聊天记录                                                              |
| `archive/k8s/`       | 历史 K8s 方案、旧服务规划和实验资料                                       |

## 操作原则

- 更新不允许直接操作服务器，必须使用 GitOps 方法进行更新。
- 集群与应用变更应先修改仓库中的声明式配置，再通过 GitOps 流程同步到目标环境。

## K3s 远端检查经验

- master 节点优先通过 Tailscale 地址 `ssh root@100.100.1.2` 访问，避免依赖固定内网地址。
- 远端非交互 SSH 环境下，`kubectl` / `k3s` 可能不在 `PATH` 中；执行集群只读检查时优先使用绝对路径 `/usr/local/bin/kubectl`。
- 服务器侧只做只读检查，不直接执行 `kubectl apply`、`kubectl edit`、`helm upgrade` 等会改变集群状态的命令。
- GitOps 变更推送后，ArgoCD 同步可能不是立即完成；`OutOfSync`、`Missing`、`Progressing` 或 `Running` 代表仍在收敛中，不应当作完成状态。
- ArgoCD 应用完成收敛时通常应看到 `Synced Healthy Succeeded`。
