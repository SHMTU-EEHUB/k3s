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
