# Mihomo / Clash

当前对应 `master1` 上的 `mihomo`、`metacubexd-ui`、`clash-proxy`、`mihomo-proxy-nodeport`、`mihomo-api-ui`、`metacubexd-ui-svc`。

## 敏感信息

完整 Mihomo `config.yaml` 已改为 SealedSecret 管理：

- 提交文件：`secret-sealed.yaml`
- 解封后 Secret：`mihomo-config`
- Secret key：`config.yaml`
- Deployment 挂载来源：`Secret/mihomo-config`

`configmap.example.yaml` 与 `config.private.example.yaml` 只作为本地生成 / 维护示例，不参与 `kustomization.yaml`。

## 当前运行口径

- 命名空间：`default`
- Workload：`Deployment/mihomo`，`Deployment/metacubexd-ui`
- Mixed 代理：`mihomo-proxy-nodeport`，NodePort `30789`
- 控制器 API：`mihomo-api-ui`，NodePort `30910`
- Web UI：`metacubexd-ui-svc`，NodePort `30911`
- Headless Service：`clash-proxy`

控制器 API 和 Web UI 只应在可信 LAN / Tailscale 内访问，不能直接暴露公网。
