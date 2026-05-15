# AI Services

当前为 `ai-services` 命名空间下的一组 AI 相关服务，统一纳管为一个应用组。

## 包含服务

- `StatefulSet/ai-postgresql`：AI 服务组共享 PostgreSQL，当前使用 `longhorn-fast-1replica`。
- `Deployment/metapi`：中转站聚合、自动签到与统一代理入口。
- `Deployment/aether`：Rust Pioneer 版 AI 网关，当前作为新的统一入口。
- `Deployment/aether-redis`：Aether 专用 Redis。
- `Deployment/ds2api`：DeepSeek Web 转 OpenAI / Claude / Gemini 兼容 API，中间件仅内部访问。
- `Deployment/kiro-rs`：Kiro 反代，走 Mihomo clean 节点。
- `Deployment/cli-proxy-api`：Codex / CLIProxyAPI 反代，走 Mihomo 默认节点。
- `Deployment/grok2api`：Grok Web 转 OpenAI / Anthropic 兼容 API，默认仅内部访问，走 Mihomo 默认节点。
- `Deployment/gpt-load`：GPT-Load 多渠道 AI 代理，走 Mihomo 默认节点，使用共享 PostgreSQL。
- `Deployment/codex2api`：Codex2API 管理台与 API 网关，走 Mihomo 默认节点，使用共享 PostgreSQL，当前启用内存缓存。

## 敏感信息

本目录使用 Sealed Secrets 管理敏感信息：

- 提交文件：`secret-sealed.yaml`
- 明文模板：`secret.example.yaml`
- 本地临时明文：只允许存在于忽略目录中，例如 `.agent-tmp/ai-services-secret.local.yaml`

涉及的 Secret：

- `ai-postgresql-secret`
- `metapi-secret`
- `aether-secret`
- `ds2api-secret`
- `kiro-rs-secret`
- `cli-proxy-api-secret`
- `grok2api-secret`
- `gpt-load-secret`
- `codex2api-secret`

## 当前运行口径

- 命名空间：`ai-services`
- 暴露方式：
  - `Service/aether` 使用 NodePort `30884`，可在可信 LAN / Tailscale 内通过 `100.100.1.2:30884` 访问。
  - Traefik Ingress 仍提供域名入口：`ai.eehub.mingz.top` → `Service/aether:8084`，`metaapi.eehub.mingz.top` → `Service/metapi:4000`。
  - `ai.eehub.mingz.top` 默认同时保留 HTTP / HTTPS；`metaapi.eehub.mingz.top` 暂时保持 HTTP。
  - `metapi`：除 Ingress 外不暴露 NodePort，`Service/metapi:4000` 保持 ClusterIP。
  - `ds2api`：仅内部访问，`Service/ds2api:5001`。
  - `grok2api`：仅内部访问，`Service/grok2api:8000`。
- 出站代理：
  - `aether`：默认 Mihomo 节点 `7897`
  - `metapi`：默认 Mihomo 节点 `7897`
  - `ds2api`：默认 Mihomo 节点 `7897`
  - `kiro-rs`：clean Mihomo 节点 `7896`
  - `cli-proxy-api`：默认 Mihomo 节点 `7897`
  - `grok2api`：默认 Mihomo 节点 `7897`
  - `gpt-load`：默认 Mihomo 节点 `7897`
  - `codex2api`：默认 Mihomo 节点 `7897`
- 数据持久化：
  - PostgreSQL：`20Gi`，`longhorn-fast-1replica`
  - Aether Redis 数据 PVC：`2Gi`，`longhorn-hdd-1replica`
  - DS2API 数据 PVC：`2Gi`，`longhorn-hdd-1replica`
  - Metapi 数据 PVC：`2Gi`，`longhorn-hdd-1replica`
  - Kiro 配置 PVC：`1Gi`，`longhorn-hdd-1replica`
  - CPA 数据 PVC：`2Gi`，`longhorn-hdd-1replica`
  - Grok2API 数据 PVC：`5Gi`，`longhorn-hdd-1replica`
  - GPT-Load 数据 PVC：`5Gi`，`longhorn-hdd-1replica`
  - Codex2API 数据 PVC：`5Gi`，`longhorn-hdd-1replica`

## 初始化说明

- PostgreSQL 首次在空数据目录启动时，会执行 `ConfigMap/ai-postgresql-init` 中的脚本，创建 `metapi` 数据库和对应用户。
- `metapi` 额外挂载 `/app/data`，用于保留本地运行数据与非数据库文件状态。
- `aether` 额外使用 initContainer 幂等创建 / 修正 `aether` 数据库与用户，兼容当前已运行的共享 PostgreSQL。
- `aether` 当前固定使用上游 `ghcr.io/fawney19/aether:0.7.0`，对应 Rust Pioneer 路线的 `v0.7.0` 正式版本。
- `ds2api` 通过 initContainer 首次将 Secret 中的 `config.json` 复制到 PVC，后续运行期 token 刷新写回 `/data/config.json`，避免重启后丢失状态。
- `kiro-rs` 通过 initContainer 将 `config.json` 与初始 `credentials.json` 复制到可写 PVC，避免 Token 刷新后无法回写。
- `cli-proxy-api` 通过 initContainer 将 `config.yaml` 复制到 PVC，并初始化持久化 `auths` 目录。
- `grok2api` 通过 initContainer 首次将 Secret 中的 `config.toml` 复制到 PVC；后续运行时配置、SQLite 账号库与日志都保存在同一个持久化卷中。
- 如需强制刷新 `grok2api` 的初始配置，需要同时更新 `grok2api-secret` 中的 `bootstrap-version`，这样 Pod 重建后会重新覆盖 PVC 内的 `config.toml`。
- `gpt-load` 使用 initContainer 幂等创建 `gpt_load` 数据库与用户；运行日志保存在 `/app/data/logs`。
- `codex2api` 使用 initContainer 幂等创建 `codex2api` 数据库与用户；当前按 PostgreSQL + 内存缓存运行，不额外引入 Redis，运行期图片与日志保存在 `/data`。
