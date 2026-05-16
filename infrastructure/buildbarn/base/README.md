# BuildBarn base

当前仓库使用 BuildBarn 作为 Buck2 / REAPI 的分布式编译与缓存基础设施。

## 当前部署口径

- `frontend`：对外 REAPI / gRPC 入口，通过 `build.eehub.mingz.top` 暴露。
- `scheduler`：仅集群内使用，负责调度执行队列与 worker 注册。
- `storage`：共享持久层，使用 `longhorn-fast-1replica`，总容量 `200Gi`。
- `worker-rust`：executor DaemonSet，只在带 `eehub.mingz.top/buildbarn-worker=true` 的节点上运行。

## 两层缓存模型

- L0：worker 本地 `emptyDir` 热缓存（`/worker/cache`）。
- L1：BuildBarn 共享 CAS / AC / FSAC，运行在 `longhorn-fast-1replica` 上。

当前设计不再依赖 MinIO / S3，也不暴露 public browser UI。

## 当前节点放置

- `master1`：`frontend` + `scheduler`
- `worker1.eehub.mingz.top`：`storage` + `worker-rust`

其中 `storage` 目前直接绑定到 `worker1.eehub.mingz.top`，因为当前 fast 层可用容量主要在该节点。

## 对外暴露

- `build.eehub.mingz.top` 只承载 BuildBarn frontend 的编译 / cache gRPC 流量。
- 不暴露 `bb-browser`，也不复用旧的编译入口域名。

Traefik 侧使用 `IngressRoute` + `scheme: h2c`，TLS 证书由 `Certificate/build-eehub-mingz-top` 与 `ClusterIssuer/letsencrypt-prod` 负责。

## 上线前检查

- `build.eehub.mingz.top` 已指向当前集群 Traefik 入口。
- `worker1.eehub.mingz.top` 已打上 `eehub.mingz.top/buildbarn-worker=true`。
- `longhorn-fast-1replica` 当前可提供至少 `200Gi` 的单盘可用空间。
- `buck2` 客户端验证会在后续单独进行；本 base 先负责把 REAPI 服务跑起来。
