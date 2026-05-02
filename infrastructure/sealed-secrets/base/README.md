# Sealed Secrets

当前集群使用 Helm 安装 Bitnami Sealed Secrets Controller。

## 版本

- Helm chart：`sealed-secrets/sealed-secrets` `2.18.5`
- Controller app：`0.36.6`
- Release：`sealed-secrets`
- Namespace：`kube-system`
- Controller name：`sealed-secrets-controller`

## 远端安装口径

远端 `master1` 已使用 Helm 安装：

- Repo：`https://bitnami-labs.github.io/sealed-secrets`
- Values：`infrastructure/sealed-secrets/base/values.yaml`

如果需要重放安装，使用相同 chart version 和 values 文件执行 `helm upgrade --install`。

## 本地加密口径

`clusters/master-node/sealed-secrets/pub-cert.pem` 是当前集群 Sealed Secrets 的公开证书，可以提交到仓库，用于本地离线加密。

明文 Secret 只允许放在本地临时文件或 `*.private.yaml`，不得提交。
