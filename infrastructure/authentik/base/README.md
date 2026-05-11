# Authentik

当前由 ArgoCD child Application 安装 Authentik Helm Chart，并把非敏感配置与加密后的 Secret 一起纳入仓库。

## 当前 GitOps 口径

- Child Application：`clusters/master-node/authentik-app.yaml`
- Chart：`authentik/authentik`
- Namespace：`authentik`
- Release：`authentik`
- Values：`infrastructure/authentik/base/values.yaml`
- Secret：`infrastructure/authentik/base/secret-sealed.yaml` → `Secret/authentik-config`

## 当前配置口径

- 入口域名：`auth.eehub.mingz.top`
- Ingress Controller：Traefik
- Ingress 入口：`web,websecure`
- 数据库：Chart 内置 PostgreSQL，持久化到 `longhorn-fast-1replica`
- Secret 管理：Sealed Secrets

## ArgoCD 同步方式

1. Root Application `clusters/master-node/sync-app.yaml` 同步 `clusters/master-node`。
2. `clusters/master-node/authentik-app.yaml` 由 ArgoCD 自动创建并接管 Helm release。
3. `infrastructure/authentik/base/` 只保留 bootstrap namespace、SealedSecret 与 values。
4. 首次初始化入口：

   - `https://auth.eehub.mingz.top/if/flow/initial-setup/`
   - 末尾 `/` 不能省略。

## Secret 维护

当前提交的 `secret-sealed.yaml` 已包含一组随机生成的初始化密钥，只保存了加密后的 SealedSecret，不包含明文。

由于当前 Helm values 使用了 `authentik.existingSecret.secretName: authentik-config`，Authentik 运行时会直接从这个 Secret 读取环境变量。因此除了密码类字段，还必须把 PostgreSQL 连接参数一并写进 Secret，否则会退回默认 `localhost:5432`。

如需轮换，请在本地基于 `secret.example.yaml` 生成新的明文 Secret，再重新封装为 `secret-sealed.yaml`：

```bash
kubeseal --format yaml --scope strict \
  --cert clusters/master-node/sealed-secrets/pub-cert.pem \
  < infrastructure/authentik/base/secret.private.yaml \
  > infrastructure/authentik/base/secret-sealed.yaml
```

建议本地 `secret.private.yaml` 至少包含以下键：

- `AUTHENTIK_SECRET_KEY`
- `AUTHENTIK_POSTGRESQL__HOST`
- `AUTHENTIK_POSTGRESQL__NAME`
- `AUTHENTIK_POSTGRESQL__USER`
- `AUTHENTIK_POSTGRESQL__PORT`
- `AUTHENTIK_POSTGRESQL__PASSWORD`
- `postgres-password`
- `password`

`secret.example.yaml` 只作为模板，不参与 `kustomization.yaml`。
