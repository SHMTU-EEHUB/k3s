# Status Page

`status-page` 命名空间部署 Gatus 状态页，用于集中观察集群内代理出口和 AI 服务组可用性。

## 分组

- `proxy`：通过 `https://www.gstatic.com/generate_204` 验证 Mihomo 默认、GPT、US、JP 四个显式代理出口，预期 HTTP 状态码为 `204`；同时通过 `mihomo-api-ui:9097` TCP 检查 Mihomo Pod / Service 是否可连通。
- `ai-services`：覆盖 `apps/ai-services/base` 中已有 Service。PostgreSQL、Redis、Kiro RS、CLIProxyAPI、WA App gRPC 使用 TCP 连通性检查；其余 HTTP 服务使用现有 readiness / liveness 路径。

## 访问入口

- 域名：`https://status.eehub.mingz.top`
- Ingress：Traefik + cert-manager `letsencrypt-prod`
- 后端：`Service/gatus:8080`
- DNS：当前由 `*.eehub.mingz.top` 通配记录覆盖，`status.eehub.mingz.top` 应解析到集群入口 IP。
- 网络口径：与 `apps/cloudflare-ddns/base` 一致，当前按校内网 DNS-only / 伪 DDNS 使用，不开启 Cloudflare 代理。

## 数据持久化

Gatus 使用 SQLite，数据库路径为 `/data/gatus.db`，由 `PersistentVolumeClaim/gatus-data` 持久化。PVC 使用 `longhorn-hdd-1replica`，容量 `1Gi`。

Deployment 设置了 `fsGroup: 1000`，用于确保 Gatus 进程可以写入 Longhorn 挂载的 `/data` 目录。后续修改 `ConfigMap/gatus-config` 时，同时递增 Pod template annotation `eehub.mingz.top/gatus-config-revision`，让 ArgoCD 同步后触发 Pod 重建并加载新配置。

UI 默认按 `group` 排序，便于直接看到 `proxy` 和 `ai-services` 两个分组。

Mihomo 外部代理出口探测间隔为 `5m`，避免一直通过节点高频访问外网；Mihomo Pod / Service 自身 TCP 存活检查仍为 `1m`，且不走代理、不产生外部流量。

## 验证

本地静态检查：

```powershell
python -c "import pathlib, yaml; cfg=yaml.safe_load(yaml.safe_load(pathlib.Path('apps/status-page/base/configmap.yaml').read_text(encoding='utf-8'))['data']['config.yaml']); print(len(cfg['endpoints']))"
Resolve-DnsName status.eehub.mingz.top
```

渲染检查：

```powershell
kubectl kustomize apps/status-page/base
kubectl kustomize clusters/master-node
```

同步后检查：

```powershell
kubectl -n status-page get pods,svc,pvc,ingress
kubectl -n status-page rollout status deployment/gatus
kubectl -n status-page get certificate,secret
kubectl -n status-page logs deployment/gatus --tail=100
```

页面与 API 检查：

```powershell
Invoke-WebRequest https://status.eehub.mingz.top/health -UseBasicParsing
Invoke-RestMethod https://status.eehub.mingz.top/api/v1/endpoints/statuses
```

如果外部域名还未接通，可先在集群侧做端口转发：

```powershell
kubectl -n status-page port-forward svc/gatus 8080:8080
Invoke-WebRequest http://127.0.0.1:8080/health -UseBasicParsing
Invoke-RestMethod http://127.0.0.1:8080/api/v1/endpoints/statuses
```
