# Cloudflare remotely-managed tunnel

This base runs a connector for the already-created, Dashboard-managed tunnel
`aether-api`. GitOps manages only the `cloudflared` connector; the tunnel's
ingress routing is remotely managed by Cloudflare and is not duplicated in a
local `config.yaml`.

## Secret

The committed `secret-sealed.yaml` produces a Secret with the exact name
`cloudflare-aether-api-tunnel-secret` and key `TUNNEL_TOKEN`, which the
Deployment references. Use `secret.example.yaml` only as a schema when rotating
and resealing the token. Never commit the unsealed Secret or real token.

## Remotely managed route

Cloudflare Dashboard stores the route for this token tunnel:

- hostname: `api.mingz.top`
- path regex: `^/v1($|/)`
- HTTP origin: `aether.ai-services.svc.cluster.local:8084`

Using `api.mingz.top` keeps the public hostname within Cloudflare Universal
SSL coverage and avoids deep-subdomain certificate limitations.

Because `cloudflared tunnel run --token` receives remotely managed ingress
configuration, this base intentionally has no ingress ConfigMap, `--config`
argument, or config volume. Route changes must be made in Cloudflare Dashboard;
changing this Deployment does not change the published route.

## Runtime prerequisites

- Cluster DNS must resolve `aether.ai-services.svc.cluster.local` from the
  connector Pod in the `default` namespace.
- `Service/aether` must exist in namespace `ai-services`, expose TCP port
  `8084`, and have ready endpoints.
- Any NetworkPolicy or egress control must allow the connector to reach that
  Service and Cloudflare's tunnel edge.
- The metrics listener on container port `2000` serves `/ready` for Kubernetes
  readiness and liveness probes; it is not exposed by a Kubernetes Service.
