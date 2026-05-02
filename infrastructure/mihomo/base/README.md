# Mihomo / Clash

当前对应 `master1` 上的 `mihomo`、`metacubexd-ui`、`clash-proxy`、`mihomo-proxy-nodeport`、`mihomo-api-ui`、`metacubexd-ui-svc`。

## 敏感信息

`configmap.yaml` 是安全占位配置，不包含真实订阅链接、节点密码、UUID、私钥或控制器密钥。

上线前必须用私有配置替换 `clash-config` 中的 `config.yaml`。推荐方式：

- 私有 overlay：在不提交仓库的 `config.private.yaml` 中覆盖 `clash-config`。
- Sealed Secrets / External Secrets：把完整 `config.yaml` 放入加密 Secret，然后把 Deployment 的卷来源从 ConfigMap 改为 Secret。

## 当前运行口径

- 命名空间：`default`
- Workload：`Deployment/mihomo`，`Deployment/metacubexd-ui`
- Mixed 代理：`mihomo-proxy-nodeport`，NodePort `30789`
- 控制器 API：`mihomo-api-ui`，NodePort `30910`
- Web UI：`metacubexd-ui-svc`，NodePort `30911`
- Headless Service：`clash-proxy`

控制器 API 和 Web UI 只应在可信 LAN / Tailscale 内访问，不能直接暴露公网。
