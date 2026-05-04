# Mihomo / Clash

当前对应 `master1` 上的 `mihomo`、`metacubexd-ui`、`clash-proxy`、`mihomo-proxy-nodeport`、`mihomo-clean-provider-1`、`mihomo-api-ui`、`metacubexd-ui-svc`。

Mihomo 管理面保留两个 NodePort：`mihomo-api-ui` 控制器 API `30910`，`metacubexd-ui-svc` Web UI `30911`；Mixed 代理保持 ClusterIP / Headless，不再暴露 NodePort。

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
- Mixed 代理：`mihomo-proxy-nodeport`，ClusterIP Service，端口 `7897`，仅供集群内显式代理使用。
- Clean 节点专用代理：`mihomo-clean-provider-1`，ClusterIP Service，Mihomo 监听 `7896`。
- TUN 透明网关：Mihomo Pod 挂载 `/dev/net/tun`，用于路由注入 fallback 场景。
- 控制器 API：`mihomo-api-ui`，NodePort `30910`。
- Web UI：`metacubexd-ui-svc`，NodePort `30911`。
- Web UI 域名入口：`mihomoui.eehub.mingz.top` → `Service/metacubexd-ui-svc:80`，由 Traefik Ingress 暴露。
- Headless Service：`clash-proxy`

`clean-provider-1` 在私有 Mihomo `config.yaml` 中作为 clean 节点专用代理组使用，并用独立 `listeners` 入口把 `7896` 转到该代理组；示例见 `config.private.example.yaml`。

TUN 只作为不支持显式代理的业务 Pod 的路由注入 fallback，不使用 `hostNetwork`，不会修改宿主机网络命名空间。

控制器 API 和 Web UI 只应在可信 LAN / Tailscale 内通过 NodePort 访问，不能直接暴露公网；clean 节点专用代理与 Mixed 代理仅供集群内其它容器访问。
