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
- 系统盘默认 Longhorn 磁盘：当前已启用调度，作为 PostgreSQL 迁移前的临时 SSD 层。
- 系统盘 Longhorn 路径：`/var/lib/longhorn/`
- 系统盘临时标签：`pg-fast`
- HDD 数据盘：`/mnt/storage/sdb`、`/mnt/storage/sdc`
- HDD Disk Tag：`HDD`
- 每块 HDD 预留：`50Gi`

## StorageClass

- `longhorn-pg-fast-1replica`：当前 PostgreSQL 迁移友好类，先落在 `master1:/var/lib/longhorn/`，未来可迁到 `worker1` SSD。
- `longhorn-hdd-1replica`：当前单节点阶段使用。
- `longhorn-hdd-2replica`：Worker 上线后给大容量普通持久化数据使用。
- `longhorn-hdd-3replica`：Worker 上线后给关键持久化服务使用。
- `longhorn-fast-1replica`：未来 `worker1` 的 NVMe / SSD 高速层使用。

Worker 未上线前，不要把关键数据只依赖 Longhorn 单副本保存。

## PostgreSQL 迁移路径

当前策略：先把 `master1:/var/lib/longhorn/` 作为临时 SSD 层，给 PostgreSQL 使用 `longhorn-pg-fast-1replica`。

未来 `worker1` 独立 SSD 上线后：

1. 将 `worker1` 加入 K3s 与 Longhorn。
2. 把 `worker1` 的 SSD 加入 Longhorn，并打同样的磁盘标签：`pg-fast`。
3. 将 PostgreSQL 卷副本数从 `1` 临时提升到 `2`。
4. 等待 Longhorn 在新 SSD 上完成副本 rebuild，卷状态变为 `Healthy`。
5. 对 `master1:/var/lib/longhorn/` 上的旧副本执行驱逐，使数据只保留在新 SSD。
6. 如果最终仍要单副本，再把卷副本数从 `2` 降回 `1`。
7. 如需获得真正本地 SSD I/O，再把 PostgreSQL Pod 调度到 SSD 所在节点。

注意：存储副本迁移可以接近无缝，但 PostgreSQL Pod 真正迁到新节点时通常仍会有一次短暂重建 / 重挂载窗口。
