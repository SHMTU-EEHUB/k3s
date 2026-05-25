# InvenTree

`inventory` 命名空间中的 InvenTree 电子库存 / 出入库 / BOM 管理服务。

## 包含服务

- `StatefulSet/inventree-postgresql`：InvenTree PostgreSQL 17，当前使用 `longhorn-fast-1replica`。
- `Deployment/inventree-redis`：Redis 缓存 / 后台任务队列，当前使用 `longhorn-hdd-1replica`。
- `Deployment/inventree-server`：InvenTree Web / API 服务，镜像 `inventree/inventree:stable`。
- `Deployment/inventree-worker`：InvenTree 后台 worker，命令 `invoke worker`。
- `Deployment/inventree-proxy`：Caddy 代理，负责 `/static` 与 `/media` 文件服务，其余请求反代到 `inventree-server`。
- `Ingress/inventree-web`：`https://inventory.eehub.mingz.top`，通过 Traefik + cert-manager 暴露。

## 敏感信息

本目录使用 Sealed Secrets 管理敏感信息：

- 提交文件：`secret-sealed.yaml`
- 明文模板：`secret.example.yaml`
- 本地临时明文：`.agent-tmp/inventree-secret.local.yaml`，该目录已被 `.gitignore` 忽略

涉及的 Secret：

- `inventree-postgresql-secret`：`POSTGRES_DB`、`POSTGRES_USER`、`POSTGRES_PASSWORD`、`INVENTREE_DB_PASSWORD`
- `inventree-redis-secret`：`REDIS_PASSWORD`
- `inventree-secret`：`INVENTREE_SECRET_KEY`、`INVENTREE_ADMIN_USER`、`INVENTREE_ADMIN_PASSWORD`、`INVENTREE_ADMIN_EMAIL`

重新生成 SealedSecret：

```powershell
kubeseal --format yaml --scope strict \
  --cert clusters/master-node/sealed-secrets/pub-cert.pem \
  < .agent-tmp/inventree-secret.local.yaml \
  > apps/inventree/base/secret-sealed.yaml
```

不要提交 `.agent-tmp/inventree-secret.local.yaml` 或任何未加密 Secret。

## 当前运行口径

- 命名空间：`inventory`
- 域名：`inventory.eehub.mingz.top`
- 外部入口：Traefik Ingress -> `Service/inventree-proxy:8080`
- 内部服务：`Service/inventree-server:8000`
- PostgreSQL：`20Gi`，`longhorn-fast-1replica`
- Redis：`2Gi`，`longhorn-hdd-1replica`
- InvenTree 数据卷：`20Gi`，`longhorn-hdd-1replica`
- 数据目录：`/home/inventree/data`
- 媒体文件：`/home/inventree/data/media`
- 静态文件：`/home/inventree/data/static`
- 备份目录：`/home/inventree/data/backup`

当前集群仍处于单节点 Longhorn 阶段，`longhorn-hdd-1replica` 只适合 PoC / 初期试运行。真实库存数据上线前，需要增加独立数据库 dump 与媒体文件备份。

## 部署与检查

渲染检查：

```powershell
kubectl kustomize apps/inventree/base
kubectl kustomize clusters/master-node
```

ArgoCD 同步后检查：

```powershell
kubectl -n inventory get pods,svc,pvc,ingress
kubectl -n inventory rollout status statefulset/inventree-postgresql
kubectl -n inventory rollout status deployment/inventree-redis
kubectl -n inventory rollout status deployment/inventree-server
kubectl -n inventory rollout status deployment/inventree-worker
kubectl -n inventory rollout status deployment/inventree-proxy
```

首次启动时 `INVENTREE_AUTO_UPDATE=True` 会执行 InvenTree 更新流程。初始管理员账号来自 `inventree-secret` 中的 `INVENTREE_ADMIN_*`，当前明文只保存在本地 `.agent-tmp/inventree-secret.local.yaml`。

如需手工创建管理员：

```powershell
kubectl -n inventory exec deploy/inventree-server -- invoke superuser
```

## 后续集成

- KiCad：优先评估 `inventree-kicad-plugin`，通过 InvenTree API / HTTP Library 暴露元器件元数据。
- 立创 / LCSC：优先走 LCSC 编码、BOM CSV、Ki-nTree 或 InvenTree 插件导入同步。
- 设备资产：如后续需要借还责任、资产标签与人员归属，可以再评估 Snipe-IT 与 InvenTree 并行。
