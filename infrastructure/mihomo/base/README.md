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

## 持久化边界

Mihomo 配置与运行时数据分开管理：

- 配置文件仍由 `Secret/mihomo-config` 提供，`config.yaml` 通过 `subPath: config.yaml` 只读挂载到 `/root/.config/mihomo/config.yaml`。
- 运行时目录由 `PersistentVolumeClaim/mihomo-runtime` 持久化，命名空间为 `default`。
- PVC 使用 `storageClassName: longhorn-hdd-1replica`，访问模式为 `ReadWriteOnce`，容量为 `2Gi`。
- PVC 挂载到 `/root/.config/mihomo`，保存 Mihomo 自动下载或生成的运行文件。
- 持久化范围包括 `proxy_provider/`、`rule_provider/`、`cache.db`、`geoip.dat`、`geoip.metadb`、`geosite.dat`，以及其它下载得到的 `.mrs` / `.mmdb` geodata 与缓存文件。

GitOps 通过 `infrastructure/mihomo/base/kustomization.yaml` 应用该 PVC 与挂载关系。不要把 provider、规则缓存、geodata 或其它运行缓存手工复制进 Git。

首次按该变更滚动后，`mihomo-runtime` 会是空 PVC，除非在 GitOps 之外手工迁移旧数据。Mihomo 应能重新下载或生成 provider、规则、缓存与 geodata 文件。

## 当前运行口径

- 命名空间：`default`
- Workload：`Deployment/mihomo`，`Deployment/metacubexd-ui`
- 配置 Secret：`mihomo-config`，只读挂载 `config.yaml`。
- 运行数据 PVC：`mihomo-runtime`，Longhorn HDD 单副本，挂载到 `/root/.config/mihomo`。
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
