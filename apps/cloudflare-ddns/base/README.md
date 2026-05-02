# Cloudflare DDNS

当前对应 `master1` 上的 `cloudflare-ddns` Deployment。

## 敏感信息

Cloudflare 私有材料已改为 SealedSecret 管理：

- 提交文件：`secret-sealed.yaml`
- 解封后 Secret：`cloudflare-ddns-secret`
- Secret keys：`CF_API_TOKEN`、`ZONE_ID`

普通 `ConfigMap/cloudflare-ddns-config` 只保存非敏感运行参数，例如域名、代理开关和本地网卡名。

## 当前运行口径

- 命名空间：`default`
- Workload：`Deployment/cloudflare-ddns`
- 镜像：`favonia/cloudflare-ddns:latest`
- 副本数：`1`
- 调度：固定到控制平面节点
- 更新周期：`@every 5m`
