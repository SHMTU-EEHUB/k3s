# proxy base

Kustomize base for proxy-related infrastructure that is not part of the existing Mihomo deployment.

## Resin

Resin runs in the `proxy` namespace as a single-replica smart proxy gateway.

- Image: `ghcr.io/resinat/resin:1.1.1`
- Port: `2260`
- Service DNS: `resin.proxy.svc.cluster.local:2260`
- Exposure: internal `ClusterIP` only, no Ingress
- Persistent state: `PersistentVolumeClaim/resin-state` mounted at `/var/lib/resin`
- Cache/log directories use `emptyDir` and can be changed to PVCs if they need persistence.

`secret-sealed.yaml` manages `Secret/resin-secret` and contains randomly generated initial credentials. Do not commit an unencrypted Secret manifest.
