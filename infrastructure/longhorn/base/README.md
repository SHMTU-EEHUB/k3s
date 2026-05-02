# Longhorn

这里保存当前 `master1` 单节点阶段的 Longhorn 声明式配置。

## 范围

本目录不负责安装 Longhorn Helm chart；它只管理已安装 Longhorn 之后的集群内对象：

- `default-replica-count`：当前单节点阶段设为 `1`。
- `master1` 的 Longhorn Node 磁盘声明。
- 面向业务的 StorageClass 规划。

## 当前运行口径

- 命名空间：`longhorn-system`
- Longhorn 版本：`v1.11.1`
- 当前节点：`master1`
- 系统盘默认 Longhorn 磁盘：保留但禁止调度。
- HDD 数据盘：`/mnt/storage/sdb`、`/mnt/storage/sdc`
- HDD Disk Tag：`HDD`
- 每块 HDD 预留：`50Gi`

## StorageClass

- `longhorn-hdd-1replica`：当前单节点阶段使用。
- `longhorn-hdd-2replica`：Worker 上线后给大容量普通持久化数据使用。
- `longhorn-hdd-3replica`：Worker 上线后给关键持久化服务使用。
- `longhorn-fast-1replica`：未来 `worker1` 的 NVMe / SSD 高速层使用。

Worker 未上线前，不要把关键数据只依赖 Longhorn 单副本保存。
