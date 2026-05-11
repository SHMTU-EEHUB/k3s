# cert-manager

当前由 ArgoCD child Application 安装 cert-manager Helm Chart，并把 Cloudflare DNS-01 所需的 Secret 与 Let’s Encrypt ClusterIssuer 一起纳入仓库。

## 当前 GitOps 口径

- Child Application：`clusters/master-node/cert-manager-app.yaml`
- Chart：`cert-manager`
- Version：`v1.20.2`
- Namespace：`cert-manager`
- Release：`cert-manager`
- Values：`infrastructure/cert-manager/base/values.yaml`
- Secret：`infrastructure/cert-manager/base/secret-sealed.yaml` → `Secret/cloudflare-api-token`

## 当前配置口径

- DNS 提供商：Cloudflare DNS-01
- 域名范围：`eehub.mingz.top`
- ACME 邮箱：`notice@mingz.top`
- Secret 管理：Sealed Secrets
- 证书签发：同时保留 `letsencrypt-staging` 与 `letsencrypt-prod`

## ArgoCD 同步方式

1. Root Application `clusters/master-node/sync-app.yaml` 同步 `clusters/master-node`。
2. `clusters/master-node/cert-manager-app.yaml` 由 ArgoCD 自动创建并接管 Helm release。
3. `infrastructure/cert-manager/base/` 只保留 bootstrap namespace、SealedSecret、ClusterIssuer 与 values。
4. 确认 ClusterIssuer Ready：

   ```bash
   kubectl get clusterissuer
   kubectl describe clusterissuer letsencrypt-prod
   ```

## Secret 维护

当前提交的 `secret-sealed.yaml` 只保存了加密后的 Cloudflare API Token，不包含明文。

如需轮换，请在本地基于 `secret.example.yaml` 生成新的明文 Secret，再重新封装为 `secret-sealed.yaml`：

```bash
kubeseal --format yaml --scope strict \
  --cert clusters/master-node/sealed-secrets/pub-cert.pem \
  < infrastructure/cert-manager/base/secret.private.yaml \
  > infrastructure/cert-manager/base/secret-sealed.yaml
```

建议本地 `secret.private.yaml` 至少包含以下键：

- `api-token`

`secret.example.yaml` 只作为模板，不参与 `kustomization.yaml`。

## 使用说明

- 本目录依赖 cert-manager CRD，ClusterIssuer 资源会在 cert-manager child Application 就绪后由 ArgoCD 同步。
- 当前默认给 `ai.eehub.mingz.top` 与 `auth.eehub.mingz.top` 开 HTTPS。
- 为了避免一次切换导致现有入口中断，Ingress 暂时同时保留 `web` 与 `websecure`，不强制 HTTP 301 跳转。
