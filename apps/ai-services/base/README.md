# AI Services

当前为 `ai-services` 命名空间下的一组 AI 相关服务，统一纳管为一个应用组。

## 包含服务

- `StatefulSet/ai-postgresql`：AI 服务组共享 PostgreSQL，当前使用 `longhorn-fast-1replica`。
- `Deployment/metapi`：中转站聚合、自动签到与统一代理入口。
- `Deployment/aether`：Rust Pioneer 版 AI 网关，当前作为新的统一入口。
- `Deployment/aether-redis`：Aether 专用 Redis。
- `Deployment/ai-services-redis`：GPT-Load 与 Codex2API 共用 Redis。
- `Deployment/kiro-rs`：Kiro 反代，走 Mihomo clean 节点。
- `Deployment/cli-proxy-api`：Codex / CLIProxyAPI 反代，走 Mihomo 默认节点。
- `Deployment/grok2api`：Grok Web 转 OpenAI / Anthropic 兼容 API，默认仅内部访问，走 Mihomo 默认节点。
- `Deployment/gpt-load`：GPT-Load 多渠道 AI 代理，走 Mihomo 默认节点，使用共享 PostgreSQL 与共享 Redis。
- `Deployment/codex2api`：Codex2API 管理台与 API 网关，走 Mihomo 默认节点，使用共享 PostgreSQL 与共享 Redis。
- `Deployment/halowebui`：HaloWebUI AI Web 控制台，走 Mihomo 默认节点，使用共享 PostgreSQL 与共享 Redis。
- `Deployment/outlook-email`：OutlookMail Plus 邮箱管理台，默认仅内部访问，走 Mihomo 默认节点，使用本地 SQLite。

## 敏感信息

本目录使用 Sealed Secrets 管理敏感信息：

- 提交文件：`secret-sealed.yaml`
- 明文模板：`secret.example.yaml`
- 本地临时明文：只允许存在于忽略目录中，例如 `.agent-tmp/ai-services-secret.local.yaml`

涉及的 Secret：

- `ai-postgresql-secret`
- `metapi-secret`
- `aether-secret`
- `kiro-rs-secret`
- `cli-proxy-api-secret`
- `grok2api-secret`
- `ai-services-redis-secret`
- `gpt-load-secret`
- `codex2api-secret`
- `halowebui-secret`
- `outlook-email-secret`

## 当前运行口径

- 命名空间：`ai-services`
- 暴露方式：
  - `Service/aether` 使用 NodePort `30884`，可在可信 LAN / Tailscale 内通过 `100.100.1.2:30884` 访问。
  - Traefik Ingress 仍提供域名入口：`ai.eehub.mingz.top` → `Service/aether:8084`，`metaapi.eehub.mingz.top` → `Service/metapi:4000`，`chat.eehub.mingz.top` → `Service/halowebui:8080`。
  - `ai.eehub.mingz.top` 默认同时保留 HTTP / HTTPS；`metaapi.eehub.mingz.top` 暂时保持 HTTP。
  - `chat.eehub.mingz.top` 通过 Traefik + cert-manager 提供 HTTPS，并承载 HaloWebUI 的 WebSocket 路由。
  - `metapi`：除 Ingress 外不暴露 NodePort，`Service/metapi:4000` 保持 ClusterIP。
  - `grok2api`：仅内部访问，`Service/grok2api:8000`。
- 出站代理：
  - `aether`：默认 Mihomo 节点 `7897`
  - `metapi`：默认 Mihomo 节点 `7897`
  - `kiro-rs`：clean Mihomo 节点 `7896`
  - `cli-proxy-api`：默认 Mihomo 节点 `7897`
  - `grok2api`：默认 Mihomo 节点 `7897`
  - `gpt-load`：默认 Mihomo 节点 `7897`
  - `codex2api`：默认 Mihomo 节点 `7897`
  - `halowebui`：默认 Mihomo 节点 `7897`
  - `outlook-email`：默认 Mihomo 节点 `7897`
- 数据持久化：
  - PostgreSQL：`20Gi`，`longhorn-fast-1replica`
  - Aether Redis 数据 PVC：`2Gi`，`longhorn-hdd-1replica`
  - AI Services Redis 数据 PVC：`2Gi`，`longhorn-hdd-1replica`
  - Metapi 数据 PVC：`2Gi`，`longhorn-hdd-1replica`
  - Kiro 配置 PVC：`1Gi`，`longhorn-hdd-1replica`
  - CPA 数据 PVC：`2Gi`，`longhorn-hdd-1replica`
  - Grok2API 数据 PVC：`5Gi`，`longhorn-hdd-1replica`
  - GPT-Load 数据 PVC：`5Gi`，`longhorn-hdd-1replica`
  - Codex2API 数据 PVC：`5Gi`，`longhorn-hdd-1replica`
  - HaloWebUI 数据 PVC：`5Gi`，`longhorn-hdd-1replica`
  - OutlookMail Plus 数据 PVC：`5Gi`，`longhorn-hdd-1replica`

## 初始化说明

- PostgreSQL 首次在空数据目录启动时，会执行 `ConfigMap/ai-postgresql-init` 中的脚本，创建 `metapi` 数据库和对应用户。
- `metapi` 额外挂载 `/app/data`，用于保留本地运行数据与非数据库文件状态。
- `aether` 额外使用 initContainer 幂等创建 / 修正 `aether` 数据库与用户，兼容当前已运行的共享 PostgreSQL。
- `aether` 当前固定使用上游 `ghcr.io/fawney19/aether:0.7.6@sha256:a388ce990340b906dc32f60184572a052015163af3305c70db273e47291bc408`，对应 Rust Pioneer 路线的正式版本。
- `kiro-rs` 通过 initContainer 将 `config.json` 与初始 `credentials.json` 复制到可写 PVC，避免 Token 刷新后无法回写。
- `cli-proxy-api` 通过 initContainer 将 `config.yaml` 复制到 PVC，并初始化持久化 `auths` 目录。
- `grok2api` 通过 initContainer 首次将 Secret 中的 `config.toml` 复制到 PVC；后续运行时配置、SQLite 账号库与日志都保存在同一个持久化卷中。
- 如需强制刷新 `grok2api` 的初始配置，需要同时更新 `grok2api-secret` 中的 `bootstrap-version`，这样 Pod 重建后会重新覆盖 PVC 内的 `config.toml`。
- `gpt-load` 使用 initContainer 幂等创建 `gpt_load` 数据库与用户；当前按 PostgreSQL + Redis 运行，使用 `ai-services-redis` 的 database 0，并显式保留 Mihomo 默认代理，运行日志保存在 `/app/data/logs`。
- `codex2api` 使用 initContainer 幂等创建 `codex2api` 数据库与用户；当前按 PostgreSQL + Redis 缓存运行，使用 `ai-services-redis` 的 database 1，运行期图片与日志保存在 `/data`。
- `halowebui` 使用 initContainer 幂等创建 `halowebui` 数据库与用户；当前按 PostgreSQL + Redis 运行，使用 `ai-services-redis` 的 database 2，通过 `ai-services-redis-secret` 中的 default Redis 密码连接，并通过 `WEBSOCKET_MANAGER=redis` 将 WebSocket 事件同步也挂到共享 Redis。
- `halowebui` 数据目录固定为 `/app/backend/data`，用于保留上传内容与运行时缓存；`WEBUI_SECRET_KEY` 必须稳定，首次注册用户会成为管理员。
- `halowebui-secret` 只保存 HaloWebUI 自身的 `WEBUI_SECRET_KEY`、`HALOWEBUI_DB_PASSWORD` 与 `DATABASE_URL`；共享 Redis 密码继续由 `ai-services-redis-secret` 统一保存。
- OutlookMail Plus（`outlook-email`）当前固定使用 `ghcr.io/zeropointsix/outlook-email-plus:main-9a8fe77@sha256:27d410788a364fb834c88dc2fbc0ad063b31f81c2351814392796753b1d464e0`（上游 2026-05-19 发布窗口的可拉取镜像，registry 暂无 `v2.6.0` 镜像 tag），数据目录为 `/app/data`，SQLite 数据库位于 `/app/data/outlook_accounts.db`，并通过 `SECRET_KEY` 与 `LOGIN_PASSWORD` 控制初始登录与加密状态。
- OutlookMail Plus（`outlook-email`）按上游建议保持单副本 + `Recreate`，不启用 Docker socket / Watchtower 自更新；Kubernetes 中未挂载 Docker socket，健康检查使用 `/healthz`。
