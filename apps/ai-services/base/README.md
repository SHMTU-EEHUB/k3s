# AI Services

当前为 `ai-services` 命名空间下的一组 AI 相关服务，统一纳管为一个应用组。

## 包含服务

- `StatefulSet/ai-postgresql`：AI 服务组共享 PostgreSQL，当前使用 `longhorn-fast-1replica`。
- `Deployment/metapi`：中转站聚合、自动签到与统一代理入口。
- `Deployment/aether`：Rust Pioneer 版 AI 网关，当前作为新的统一入口。
- `Deployment/aether-redis`：Aether 专用 Redis。
- `Deployment/ai-services-redis`：GPT-Load、Codex2API 与 HaloWebUI 共用 Redis。
- `Deployment/kiro-rs`：Kiro 反代，走 Mihomo GPT 专用节点。
- `Deployment/cli-proxy-api`：Codex / CLIProxyAPI 反代，走 Mihomo 默认节点。
- `Deployment/grok2api`：Grok Web 转 OpenAI / Anthropic 兼容 API，默认仅内部访问，走 Mihomo 默认节点。
- `Deployment/gpt-load`：GPT-Load 多渠道 AI 代理，走 Mihomo 默认节点，使用共享 PostgreSQL 与共享 Redis。
- `Deployment/codex2api`：Codex2API 管理台与 API 网关，走 Mihomo 默认节点，使用共享 PostgreSQL 与共享 Redis。
- `Deployment/halowebui`：HaloWebUI AI Web 控制台，走 Mihomo 默认节点，使用共享 PostgreSQL 与共享 Redis。
- `Deployment/gemini-web2api`：Gemini Web 转 OpenAI 兼容 API，默认仅内部访问，走 Mihomo 默认节点，使用 ConfigMap 保存非敏感配置，不启用应用层 API key。
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

## Gemini Web2API 访问

- 集群内完整地址：`http://gemini-web2api.ai-services.svc.cluster.local:8081`
- `ai-services` 命名空间内短地址：`http://gemini-web2api:8081`
- 仅限 Kubernetes 集群内部访问：未配置 Ingress、NodePort、LoadBalancer 或 HTTPRoute；这不是“同一个容器内”限定，而是集群内 Pod / 节点可访问。
- 当前 `ConfigMap/gemini-web2api-config` 中 `api_keys: []`，上游会关闭应用层鉴权；随意填写或不填写 Bearer key 都不影响访问，因此不为 Gemini 额外配置 Secret。

## 当前运行口径

- 命名空间：`ai-services`
- 暴露方式：
  - `Service/aether` 使用 NodePort `30884`，可在可信 LAN / Tailscale 内通过 `100.100.1.2:30884` 访问。
  - Traefik Ingress 仍提供域名入口：`ai.eehub.mingz.top` → `Service/aether:8084`，`metaapi.eehub.mingz.top` → `Service/metapi:4000`，`chat.eehub.mingz.top` → `Service/halowebui:8080`。
  - `ai.eehub.mingz.top` 默认同时保留 HTTP / HTTPS；`metaapi.eehub.mingz.top` 暂时保持 HTTP。
  - `chat.eehub.mingz.top` 通过 Traefik + cert-manager 提供 HTTPS，并承载 HaloWebUI 的 WebSocket 路由。
  - `metapi`：除 Ingress 外不暴露 NodePort，`Service/metapi:4000` 保持 ClusterIP。
  - `grok2api`：仅内部访问，`Service/grok2api:8000`。
  - `gemini-web2api`：仅内部访问，`Service/gemini-web2api:8081` 保持 ClusterIP，无 Ingress / NodePort。
- 出站代理：
  - `aether`：默认 Mihomo 节点 `7897`
  - `metapi`：默认 Mihomo 节点 `7897`
  - `kiro-rs`：GPT Mihomo 节点 `mihomo-gpt-listener:7910`
  - `cli-proxy-api`：默认 Mihomo 节点 `7897`
  - `grok2api`：默认 Mihomo 节点 `7897`
  - `gpt-load`：默认 Mihomo 节点 `7897`
  - `codex2api`：默认 Mihomo 节点 `7897`
  - `halowebui`：默认 Mihomo 节点 `7897`
  - `gemini-web2api`：默认 Mihomo 节点 `7897`
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
- `aether` 当前固定使用上游 `ghcr.io/fawney19/aether:0.7.7@sha256:99c0339d2ad69fbd1ca01f98e3155176560724c1670d95e81b6a02d272d7aef0`，对应 Rust Pioneer 路线的正式版本。
- `kiro-rs` 通过 initContainer 将 `config.json` 与初始 `credentials.json` 复制到可写 PVC，避免 Token 刷新后无法回写；当前代理指向 `http://mihomo-gpt-listener.default.svc.cluster.local:7910`。
- `cli-proxy-api` 通过 initContainer 将 `config.yaml` 复制到 PVC，并初始化持久化 `auths` 目录。
- `grok2api` 通过 initContainer 首次将 Secret 中的 `config.toml` 复制到 PVC；后续运行时配置、SQLite 账号库与日志都保存在同一个持久化卷中。
- 如需强制刷新 `grok2api` 的初始配置，需要同时更新 `grok2api-secret` 中的 `bootstrap-version`，这样 Pod 重建后会重新覆盖 PVC 内的 `config.toml`。
- `gpt-load` 使用 initContainer 幂等创建 `gpt_load` 数据库与用户；当前固定使用 `ghcr.io/tbphp/gpt-load:v1.4.8@sha256:719d5c885f3f3fc228123b78f6788c8604505c3c20717635f91fbb92e63fade3`，按 PostgreSQL + Redis 运行，使用 `ai-services-redis` 的 database 0，并显式保留 Mihomo 默认代理，运行日志保存在 `/app/data/logs`。
- `codex2api` 使用 initContainer 幂等创建 `codex2api` 数据库与用户；当前固定使用 `ghcr.io/james-6-23/codex2api:2.2.6@sha256:ed866fd9e0ea9b9f659d4418c8190bf3155ce711ffbcd272ff1525530a5075c6`，按 PostgreSQL + Redis 缓存运行，使用 `ai-services-redis` 的 database 1，运行期图片与日志保存在 `/data`。
- `halowebui` 使用 initContainer 幂等创建 `halowebui` 数据库与用户；当前固定使用 `ghcr.io/ztx888/halowebui:main@sha256:b412ab407bd89c5e204031f2d0f03b4a26fa99b1fd2b98043330b810f661b5fe`，按 PostgreSQL + Redis 运行，使用 `ai-services-redis` 的 database 2，通过 `ai-services-redis-secret` 中的 default Redis 密码连接，并通过 `WEBSOCKET_MANAGER=redis` 将 WebSocket 事件同步也挂到共享 Redis。
- `halowebui` 数据目录固定为 `/app/backend/data`，用于保留上传内容与运行时缓存；`WEBUI_SECRET_KEY` 必须稳定，首次注册用户会成为管理员。
- `halowebui-secret` 只保存 HaloWebUI 自身的 `WEBUI_SECRET_KEY`、`HALOWEBUI_DB_PASSWORD` 与 `DATABASE_URL`；共享 Redis 密码继续由 `ai-services-redis-secret` 统一保存。
- `gemini-web2api` 当前固定使用 `ghcr.io/sophomoresty/gemini-web2api:latest@sha256:3012cde052f25f0d9ff4bcde7df333da228bb5e5baa637a368c0c82b153271ac`，通过 `ConfigMap/gemini-web2api-config` 挂载 `/app/config.json`，监听 `0.0.0.0:8081`，配置内保留 `api_keys: []`，不启用应用层 API key，并使用 Mihomo 默认代理 `http://mihomo-proxy-nodeport.default.svc.cluster.local:7897`。
- OutlookMail Plus（`outlook-email`）当前固定使用 `ghcr.io/zeropointsix/outlook-email-plus:v2.7.0@sha256:4b0c000675c4ec8ad36bab1f5c5e03ed32fa3c7f1ce54740eb09994ff1620d7e`，数据目录为 `/app/data`，SQLite 数据库位于 `/app/data/outlook_accounts.db`，并通过 `SECRET_KEY` 与 `LOGIN_PASSWORD` 控制初始登录与加密状态。
- OutlookMail Plus（`outlook-email`）上游标签检查结果：`v2.7.0` 截至 2026-06-02 仍为当前标签；仓库内没有明文 Secret，无法在 Git 中执行明文凭据轮换。
- OutlookMail Plus（`outlook-email`）按上游建议保持单副本 + `Recreate`，不启用 Docker socket / Watchtower 自更新；Kubernetes 中未挂载 Docker socket，健康检查使用 `/healthz`。
