# ArgoCD

当前集群使用 Helm 安装 ArgoCD，并用它同步 `clusters/master-node`。

## 建议安装口径

- Chart：`argo/argo-cd`
- Namespace：`argocd`
- Release：`argocd`
- Values：`infrastructure/argocd/base/values.yaml`

## 当前访问口径

- `argocd-server` Service 类型：`ClusterIP`

当前 values 使用 `server.insecure=true`。日常不再通过 NodePort 暴露 ArgoCD；如需临时访问，可通过 `kubectl port-forward` 或在可信网络内另行配置受控入口。

## GitHub 访问

当前 values 已给 `repoServer` 注入代理环境变量，通过集群内 `mihomo-proxy-nodeport.default.svc.cluster.local:7897` 访问 GitHub。

## GitOps 入口

仓库同步入口：`clusters/master-node/sync-app.yaml`

在 ArgoCD 已安装后，可应用该文件，让集群从 `https://github.com/SHMTU-EEHUB/k3s.git` 自动同步。
