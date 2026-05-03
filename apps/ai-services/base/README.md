# AI Services

当前为 `ai-services` 命名空间下的一组 AI 相关服务，统一纳管为一个应用组。

## 包含服务

- `StatefulSet/ai-postgresql`：AI 服务组共享 PostgreSQL，当前使用 `longhorn-fast-1replica`。
- `Deployment/axonhub`：AI 分发 / 统一网关。
- `Deployment/metapi`：中转站聚合、自动签到与统一代理入口。
- `Deployment/kiro-rs`：Kiro 反代，走 Mihomo clean 节点。
- `Deployment/cli-proxy-api`：Codex / CLIProxyAPI 反代，走 Mihomo 默认节点。

## 敏感信息

本目录使用 Sealed Secrets 管理敏感信息：

- 提交文件：`secret-sealed.yaml`
- 明文模板：`secret.example.yaml`
- 本地临时明文：只允许存在于忽略目录中，例如 `.agent-tmp/ai-services-secret.local.yaml`

涉及的 Secret：

- `ai-postgresql-secret`
- `axonhub-secret`
- `metapi-secret`
- `kiro-rs-secret`
- `cli-proxy-api-secret`

## 当前运行口径

- 命名空间：`ai-services`
- 暴露方式：服务本身默认使用 `ClusterIP`；通过 Traefik Ingress 暴露：
  - `metaapi.eehub.mingz.top` → `Service/metapi:4000`
  - `axonhub.eehub.mingz.top` → `Service/axonhub:8090`
- 出站代理：
  - `metapi`：默认 Mihomo 节点 `7897`
  - `kiro-rs`：clean Mihomo 节点 `7896`
  - `cli-proxy-api`：默认 Mihomo 节点 `7897`
- 数据持久化：
  - PostgreSQL：`20Gi`，`longhorn-fast-1replica`
  - Metapi 数据 PVC：`2Gi`，`longhorn-hdd-1replica`
  - Kiro 配置 PVC：`1Gi`，`longhorn-hdd-1replica`
  - CPA 数据 PVC：`2Gi`，`longhorn-hdd-1replica`

## 初始化说明

- PostgreSQL 首次在空数据目录启动时，会执行 `ConfigMap/ai-postgresql-init` 中的脚本，创建 `axonhub` / `metapi` 数据库和对应用户。
- `metapi` 额外挂载 `/app/data`，用于保留本地运行数据与非数据库文件状态。
- `kiro-rs` 通过 initContainer 将 `config.json` 与初始 `credentials.json` 复制到可写 PVC，避免 Token 刷新后无法回写。
- `cli-proxy-api` 通过 initContainer 将 `config.yaml` 复制到 PVC，并初始化持久化 `auths` 目录。
