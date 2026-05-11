# cert-manager

当前使用官方 Helm Chart 安装 cert-manager，并把 Cloudflare DNS-01 所需的 Secret 与 Let’s Encrypt ClusterIssuer 一起纳入仓库。

## 建议安装口径

- Chart：`oci://quay.io/jetstack/charts/cert-manager`
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

## 安装步骤

1. 先确认 Sealed Secrets Controller 已安装。
2. 使用仓库中的 values 安装 / 升级 cert-manager Helm Release：

   ```bash
   helm upgrade --install cert-manager oci://quay.io/jetstack/charts/cert-manager \
     --version v1.20.2 \
     -n cert-manager \
     --create-namespace \
     -f infrastructure/cert-manager/base/values.yaml
   ```

3. 确认 cert-manager CRD 与 Pod 已就绪：

   ```bash
   kubectl get crd | grep cert-manager
   kubectl get pods -n cert-manager
   ```

4. 再应用本目录资源：

   ```bash
   kubectl apply -k infrastructure/cert-manager/base
   ```

5. 确认 ClusterIssuer Ready：

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

- 本目录依赖 cert-manager CRD，不能在 cert-manager Helm Release 安装前直接应用。
- 当前默认给 `ai.eehub.mingz.top` 与 `auth.eehub.mingz.top` 开 HTTPS。
- 为了避免一次切换导致现有入口中断，Ingress 暂时同时保留 `web` 与 `websecure`，不强制 HTTP 301 跳转。
