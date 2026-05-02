# ArgoCD

当前集群使用 Helm 安装 ArgoCD，并用它同步 `clusters/master-node`。

## 建议安装口径

- Chart：`argo/argo-cd`
- Namespace：`argocd`
- Release：`argocd`
- Values：`infrastructure/argocd/base/values.yaml`

## 当前访问口径

- `argocd-server` Service 类型：`NodePort`
- HTTP NodePort：`30930`

当前 values 使用 `server.insecure=true`，适合先在可信 LAN / Tailscale 内快速启用。后续如需正式暴露，再接入 Ingress / TLS。

## GitOps 入口

仓库同步入口：`clusters/master-node/sync-app.yaml`

在 ArgoCD 已安装后，可应用该文件，让集群从 `https://github.com/SHMTU-EEHUB/k3s.git` 自动同步。
