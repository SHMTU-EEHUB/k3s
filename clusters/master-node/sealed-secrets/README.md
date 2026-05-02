# Sealed Secrets public certificate

`pub-cert.pem` 是当前 `master-node` 集群 Sealed Secrets Controller 的公开证书。

用途：本地用 `kubeseal --cert clusters/master-node/sealed-secrets/pub-cert.pem` 加密 Secret，生成可提交的 `SealedSecret`。

注意：这是公钥证书，不是私钥，可以提交。若远端 Controller 轮换密钥，需要重新抓取并更新本文件，同时重新生成需要更新的 SealedSecret。
