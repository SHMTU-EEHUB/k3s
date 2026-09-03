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
- `Deployment/grok2api`：官方 v3.1.5 Grok 网关，`Service/grok2api` 当前承载该版本。
- `Deployment/gpt-load`：GPT-Load 多渠道 AI 代理，走 Mihomo 默认节点，使用共享 PostgreSQL 与共享 Redis。
- `Deployment/codex2api`：Codex2API 管理台与 API 网关，走 Mihomo 默认节点，使用共享 PostgreSQL 与共享 Redis。
- `Deployment/halowebui`：HaloWebUI AI Web 控制台，走 Mihomo 默认节点，使用共享 PostgreSQL 与共享 Redis。
- `Deployment/gemini-web2api`：Gemini Web 转 OpenAI 兼容 API，默认仅内部访问，走 Mihomo 默认节点，使用 ConfigMap 保存非敏感配置，不启用应用层 API key。
- `Deployment/outlook-email`：OutlookMail Plus 邮箱管理台，默认仅内部访问，走 Mihomo 默认节点，使用本地 SQLite。
- `Deployment/copilot-api`：GitHub Copilot 转 OpenAI / Anthropic / Gemini 兼容 API，默认仅内部访问，走 Mihomo 默认节点，使用本地数据目录保存管理配置。
- `Deployment/notion2api`：Notion AI 转 OpenAI 兼容 API，默认仅内部访问，走 Mihomo 默认节点，使用本地 SQLite 与账号目录。
- `Deployment/wa-app`：WhatsApp 账号管理与 gRPC 服务，默认仅内部访问，走 Mihomo GPT 专用节点，使用本地 SQLite 数据目录。
- `Deployment/chatgpt2api`：ChatGPT 图片代理，默认仅内部访问，走 Mihomo GPT 专用节点，使用本地 JSON 配置与数据目录。

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
- `chatgpt2api-secret`

`copilot-api`、`notion2api` 与 `wa-app` 当前不新增提交到仓库的 Secret；首次配置通过 `kubectl port-forward` 访问内部服务完成。管理密码、API Key、Copilot / Notion / WhatsApp 账号 Cookie、Token 与 Refresh Token 均不得以明文提交。

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
  - `grok2api`：仅内部访问，`Service/grok2api:8000` 当前承载 v3。
  - `gemini-web2api`：仅内部访问，`Service/gemini-web2api:8081` 保持 ClusterIP，无 Ingress / NodePort。
  - `copilot-api`：仅内部访问，`Service/copilot-api:4141` 保持 ClusterIP，无 Ingress / NodePort；首次管理初始化建议使用 `kubectl -n ai-services port-forward svc/copilot-api 4141:4141` 后访问 `http://127.0.0.1:4141/admin`。
  - `notion2api`：仅内部访问，`Service/notion2api:8787` 保持 ClusterIP，无 Ingress / NodePort；首次管理初始化建议使用 `kubectl -n ai-services port-forward svc/notion2api 8787:8787` 后访问 `http://127.0.0.1:8787/admin`。
  - `wa-app`：仅内部访问，`Service/wa-app:8080` 提供 Dashboard，`Service/wa-app-grpc:50091` 提供 gRPC，无 Ingress / NodePort；首次使用建议 `kubectl -n ai-services port-forward svc/wa-app 8080:8080` 后访问 `http://127.0.0.1:8080`。
- 出站代理：
  - `aether`：默认不设置标准代理环境变量，不走 Mihomo；如需临时走代理，应显式添加 `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY` 并确保 Authentik/OIDC 令牌兑换目标绕过代理。
  - `metapi`：默认 Mihomo 节点 `7897`
  - `kiro-rs`：GPT Mihomo 节点 `mihomo-gpt-listener:7910`
  - `cli-proxy-api`：默认 Mihomo 节点 `7897`
  - `grok2api`：标准代理环境变量指向 Mihomo 节点 `7897`，仅覆盖 Build API / Statsig；Web chat、image、video 必须另在管理后台配置数据库出站 HTTP 节点。
  - `gpt-load`：默认 Mihomo 节点 `7897`
  - `codex2api`：默认 Mihomo 节点 `7897`
  - `halowebui`：默认 Mihomo 节点 `7897`
  - `gemini-web2api`：默认 Mihomo 节点 `7897`
  - `outlook-email`：默认 Mihomo 节点 `7897`
  - `copilot-api`：默认 Mihomo 节点 `7897`，并通过 `PROXY_ENV=true` 读取标准代理环境变量。
  - `notion2api`：默认 Mihomo 节点 `7897`，并通过 `N2A_PROXY_MODE=env` 读取标准代理环境变量。
  - `wa-app`：GPT Mihomo 节点 `mihomo-gpt-listener:7910`，同时设置 `WA_COMMON_PROXY` 与标准代理环境变量。
- 数据持久化：
  - PostgreSQL：`20Gi`，`longhorn-fast-1replica`
  - Aether Redis 数据 PVC：`2Gi`，`longhorn-hdd-1replica`
  - AI Services Redis 数据 PVC：`2Gi`，`longhorn-hdd-1replica`
  - Metapi 数据 PVC：`2Gi`，`longhorn-hdd-1replica`
  - Kiro 配置 PVC：`1Gi`，`longhorn-hdd-1replica`
  - CPA 数据 PVC：`2Gi`，`longhorn-hdd-1replica`
  - Grok2API 数据 PVC `grok2api-data`：`5Gi`，`longhorn-hdd-1replica`。
  - GPT-Load 数据 PVC：`5Gi`，`longhorn-hdd-1replica`
  - Codex2API 数据 PVC：`5Gi`，`longhorn-hdd-1replica`
  - HaloWebUI 数据 PVC：`5Gi`，`longhorn-hdd-1replica`
  - OutlookMail Plus 数据 PVC：`5Gi`，`longhorn-hdd-1replica`
  - Copilot API 数据 PVC：`5Gi`，`longhorn-hdd-1replica`
  - Notion2API 数据 PVC：`5Gi`，`longhorn-hdd-1replica`
  - WA App 数据 PVC：`5Gi`，`longhorn-hdd-1replica`
  - ChatGPT2API 数据 PVC：`10Gi`，`longhorn-hdd-1replica`

## 初始化说明

- PostgreSQL 首次在空数据目录启动时，会执行 `ConfigMap/ai-postgresql-init` 中的脚本，创建 `metapi` 数据库和对应用户。
- 基础镜像核验（2026-09-03）：PostgreSQL 16.15 / Alpine 3.24，index `sha256:cf78e76683b9ca8c5733cbbdce6c9262b45b6767934dd0a95e671f9a0fc20685`，`linux/amd64` `sha256:075f7ba66bc9b3ce7d6b8b635208ff61cd7cf1a67d71ec530eec5d7ae0cbe571`；官方来源：https://www.postgresql.org/docs/16/release-16-15.html。
- Redis 7.4.11 / Alpine 3.21，index `sha256:ff02b58f971e7d7d156a1267e283fcbbeee91773b6aa36c49dac28ecfe28eadf`，`linux/amd64` `sha256:1db42ccef14898aa29bae778452d567534b59c107129cbc1163fb552de184d3c`；官方来源：https://github.com/redis/redis/releases/tag/7.4.11。
- `metapi` 当前固定使用 `1467078763/metapi:sha-41767a6@sha256:d6118229e7d2423262b253a419baf18c22f1682a7bbc7d3c756d090aa2b295c6`，额外挂载 `/app/data`，用于保留本地运行数据与非数据库文件状态。
- `aether` 额外使用 initContainer 幂等创建 / 修正 `aether` 数据库与用户，兼容当前已运行的共享 PostgreSQL。
- `aether` 当前固定使用上游 `ghcr.io/fawney19/aether:0.7.13@sha256:c0f12ae4d5dcd18f0632af637ab851403b71042ee496265daabd59c4e49dedc0`（2026-08-18 发布并核验；`linux/amd64` manifest `sha256:5a321d9caa1f1ada607d6edc4d622072bc07d16e5164bbc707b28d274401d6c8`；官方 release：https://github.com/fawney19/Aether/releases/tag/v0.7.13），对应 Rust Pioneer 路线的正式版本。v0.7.13 改进 Responses 路由/兼容、模型权限与动态模型发现、候选持久化重试风暴修复，并新增 Responses WebSocket；release 未声明数据库迁移或破坏性配置变更，沿用现有数据路径/端口/启动探针。
- `aether` 的 Authentik 登录配置保存在 Aether 后台 / PostgreSQL 的 OAuth Provider 配置中，当前回调入口应使用 `https://ai.eehub.mingz.top/api/oauth/custom_authentik/callback`，前端完成页为 `https://ai.eehub.mingz.top/auth/callback`。如果出现“令牌兑换失败”，优先确认 Aether Pod 未设置 `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY` 等代理环境变量，确保 `https://auth.eehub.mingz.top/application/o/token/` 与 `/userinfo/` 不经 Mihomo 代理。
- `kiro-rs` 当前固定使用 ZyphrZero 官方二次开发版 `docker.io/zyphrzero/kiro-rs:0.8.0@sha256:118df1717fe8dc35139f05f3436a43aee7477bd4d13952c9ed35e5eb00dd079e`（2026-09-03 核验；index `sha256:118df1717fe8dc35139f05f3436a43aee7477bd4d13952c9ed35e5eb00dd079e`，`linux/amd64` `sha256:674e0d4af4f206d431ba033bcc4bac7840da73ae37a9fe0faec99ecd774f5b8c`；release：https://github.com/ZyphrZero/kiro.rs/releases/tag/v0.8.0），通过 initContainer 将 `config.json` 与初始 `credentials.json` 复制到可写 PVC，避免 Token 刷新后无法回写；旧版的 config 与 credentials 可直接沿用，当前代理指向 `http://mihomo-gpt-listener.default.svc.cluster.local:7910`。
- `cli-proxy-api` 通过 initContainer 将 `config.yaml` 复制到 PVC，并初始化持久化 `auths` 目录；当前固定使用官方 `eceasy/cli-proxy-api:v7.2.147@sha256:f077e153476466e0ea8355400e39bf1508e637585b661ed3991b7b8129ce054d`（2026-09-03 核验；index `sha256:f077e153476466e0ea8355400e39bf1508e637585b661ed3991b7b8129ce054d`，`linux/amd64` `sha256:f8d32055759ff046c87b7607117a4c92d17920e4b5be88d189ee9995a62e8ce4`；release：https://github.com/router-for-me/CLIProxyAPI/releases/tag/v7.2.147）。该版修复协议翻译、工具与插件，无数据迁移；`/CLIProxyAPI/config.yaml`、`/root/.cli-proxy-api`、端口 `8317` 与现有 PVC 保持兼容。
- `grok2api` 当前固定使用官方 `ghcr.io/chenyme/grok2api:v3.1.5@sha256:9446393fec917991919a12ca8ef9211560064c864952f0158c43694d9ca5144b`（2026-08-27 核验；index `sha256:9446393fec917991919a12ca8ef9211560064c864952f0158c43694d9ca5144b`，`linux/amd64` `sha256:854030752114d187232b959d82319fcb893a3fe1a4be3ad95a3e0062a1dc6daf`；release：https://github.com/chenyme/grok2api/releases/tag/v3.1.5）。该版修复流式、代理租约、审计、媒体协议与配额；无声明迁移，现有 `/run/grok2api/config.yaml` 配置与 `/app/data` PVC 兼容，继续固定版本而不使用 `latest`。
- 首次登录使用 `kubectl -n ai-services port-forward deployment/grok2api 8000:8000`，然后访问 `http://127.0.0.1:8000` 并以 `admin` 登录。bootstrap 密码仅保存在被忽略的 `.agent-tmp/ai-services-grok2api-secret.local.yaml` 明文清单和提交的 SealedSecret 密文中，不得提交或输出明文。
- `secrets.credentialEncryptionKey` 是已保存 provider 凭据的加密根密钥，必须保持稳定；轮换或丢失会导致既有凭据无法解密。
- v3 的标准 `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY` 仅覆盖 Build API / Statsig。Web chat、image、video 不读取这些环境变量；首次登录后必须在管理后台新增并启用指向 `http://mihomo-proxy-nodeport.default.svc.cluster.local:7897` 的数据库出站 HTTP 节点。
- `gpt-load` 使用 initContainer 幂等创建 `gpt_load` 数据库与用户；当前固定使用 `ghcr.io/tbphp/gpt-load:v1.4.10@sha256:6388081471c906796beb12d094d73db4f9d7bc73850e9ee923d0871b3476a1ae`（2026-08-27 核验；index `sha256:6388081471c906796beb12d094d73db4f9d7bc73850e9ee923d0871b3476a1ae`，`linux/amd64` `sha256:20fcca1af61d945267c23f06429050c423f21828802c644d7a53c53314064fb6`；release：https://github.com/tbphp/gpt-load/releases/tag/v1.4.10）。该版更新安全依赖并修复并发失败计数，无迁移说明；继续按 PostgreSQL + Redis 运行、使用 `ai-services-redis` database 0，保留 Mihomo 默认代理与 `/app/data/logs`。
- `codex2api` 使用 initContainer 幂等创建 `codex2api` 数据库与用户；当前固定使用 `ghcr.io/james-6-23/codex2api:2.8.9@sha256:dbbd7aca9c8f753dc2bba9a37bce59aae34b20058e13e74978cc6eafc93f44d3`（2026-09-03 核验；index `sha256:dbbd7aca9c8f753dc2bba9a37bce59aae34b20058e13e74978cc6eafc93f44d3`，`linux/amd64` `sha256:160946e7d894e6d46b427b3a349f4d1bd871f4a11801597eb68cc45f4c6d2f82`；release：https://github.com/james-6-23/codex2api/releases/tag/v2.8.9）。该版支持在线 usage index migration，无需新数据库；升级后未固定渠道的 API key 可能让 Antigravity 参与 Chat Completions/Messages 调度。端口 `8080`、`/health`、Redis 配置与 `/data` 路径保持兼容。
- `halowebui` 使用 initContainer 幂等创建 `halowebui` 数据库与用户；当前固定使用 `ghcr.io/ztx888/halowebui:main@sha256:6ac7ed58a17779f1feb7a8bc562b4546985ef478d3ea1a53ca083442b856939f`，按 PostgreSQL + Redis 运行，使用 `ai-services-redis` 的 database 2，通过 `ai-services-redis-secret` 中的 default Redis 密码连接，并通过 `WEBSOCKET_MANAGER=redis` 将 WebSocket 事件同步也挂到共享 Redis。
- `halowebui` 数据目录固定为 `/app/backend/data`，用于保留上传内容与运行时缓存；`WEBUI_SECRET_KEY` 必须稳定，首次注册用户会成为管理员。
- `halowebui-secret` 只保存 HaloWebUI 自身的 `WEBUI_SECRET_KEY`、`HALOWEBUI_DB_PASSWORD` 与 `DATABASE_URL`；共享 Redis 密码继续由 `ai-services-redis-secret` 统一保存。
- `gemini-web2api` 当前固定使用 `ghcr.io/sophomoresty/gemini-web2api:latest@sha256:abf7dfe24426c84f04ca1b3c29cfc59e40ffbb52abb5f696aebd10cfeb429055`（2026-08-14 构建，2026-08-17 核验；`linux/amd64` manifest `sha256:723691f475855793368f0a4dda976f0d08aa415c7ba2df045367be080aa036e5`），通过 `ConfigMap/gemini-web2api-config` 挂载 `/app/config.json`，监听 `0.0.0.0:8081`；配置内设置 `temporary_chats: false` 并保留 `api_keys: []`，不启用应用层 API key。`gemini_bl` 会在启动时自动刷新，当前配置值仅作为 fallback；服务使用 Mihomo 默认代理 `http://mihomo-proxy-nodeport.default.svc.cluster.local:7897`。本次构建增加分块请求体、多模态输入及 `gemini-3.7-flash` 支持，不涉及配置、端口或数据迁移。
- OutlookMail Plus（`outlook-email`）当前固定使用 `ghcr.io/zeropointsix/outlook-email-plus:v2.8.0@sha256:38ec7f0e8adbcd95277cc8e06d35cb574640eb609e1b433d1355587ea7005e09`，数据目录为 `/app/data`，SQLite 数据库位于 `/app/data/outlook_accounts.db`，并通过 `SECRET_KEY` 与 `LOGIN_PASSWORD` 控制初始登录与加密状态。
- OutlookMail Plus（`outlook-email`）上游标签检查结果：`v2.8.0` 为 2026-05-29 发布的稳定旧前端版本，启动时会将 SQLite schema 从 v23 迁移至 v24；仓库内没有明文 Secret，无法在 Git 中执行明文凭据轮换。
- OutlookMail Plus（`outlook-email`）按上游建议保持单副本 + `Recreate`，不启用 Docker socket / Watchtower 自更新；Kubernetes 中未挂载 Docker socket，健康检查使用 `/healthz`。
- Copilot API（`copilot-api`）当前固定使用 `ghcr.io/qlhazycoder/copilot-api:5.0.0@sha256:c5c998a55ab2440341e8217d4e9f5e97e52f23c943e990d51516065b262b9a39`，监听 `4141`，数据目录固定为 `/data`，健康检查使用 `/`。首次启动后通过内部端口转发访问 `/admin` 完成管理初始化，不在 Git 中提交 `ADMIN_SECRET` 或 `ADMIN_SECRET_HASH` 明文。
- Notion2API（`notion2api`）当前固定使用 `ghcr.io/galiais/notion2api:v1.0.8@sha256:7aceac16c33689ed0801b4e8798786b36a3f0ba76b14c1550e5e8c0b6ee39723`，监听 `8787`，配置文件写入 `/app/data/config.json`，SQLite 与账号目录保存在同一 PVC。首次启动会从镜像内默认配置复制配置文件，必须通过端口转发进入 `/admin` 后立即修改默认管理密码、API Key 与 Notion 账号配置。
- ChatGPT2API（`chatgpt2api`）当前固定使用 `ghcr.io/basketikun/chatgpt2api:v1.8.0@sha256:fbee934fd5363ef6ce5f7632df854f87c770e382bf0f874f8c59372eeded6aab`，监听 `80`，现有本地 JSON 存储与挂载无需迁移，`CHATGPT2API_AUTH_KEY` 由 `chatgpt2api-secret` 提供，并通过 `CHATGPT2API_PROXY` 与标准代理环境变量走 GPT Mihomo 节点。
- WA App（`wa-app`）当前固定使用 `ghcr.io/pood1e/wa-app-service:latest@sha256:57963ad850cb2e41a22cb079912723c9984dca36dbf56a293df08746afe34c8d`，gRPC 监听 `50091`，Dashboard 监听 `8080`，数据目录固定为 `/var/lib/wa-app`，健康检查使用 `/healthz`。当前不接共享 PostgreSQL / Redis，按上游默认 SQLite 运行，并通过 `WA_COMMON_PROXY` 和标准代理环境变量走 GPT Mihomo 节点。
- 不更新：Metapi `sha-f4114ad` 因非正式 Release 且 PostgreSQL schema migration 阻塞；Outlook v2.9.10 因 SQLite v25 migration 与重复规范化 API key 名预检阻塞；GPT-Load 2.0 RC 因 1.x 数据不兼容不更新。
- 截至 2026-09-03，Gemini `latest`、HaloWebUI `main`、WA `latest` 的 registry digest 未变化，均按 `tag+digest` 跟踪滚动标签；Aether、Metapi、Kiro、Copilot、Notion 与 ChatGPT 当前已是可确认支持的最新版本。
