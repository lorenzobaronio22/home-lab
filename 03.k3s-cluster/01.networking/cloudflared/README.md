# Cloudflare Tunnel Agent

Remotely-managed Cloudflare Tunnel connecting the `oci` k3s cluster to Cloudflare's edge, following
Cloudflare's [Kubernetes deployment guide](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/deployment-guides/kubernetes/).
Runs as a standalone pod (no Tailscale sidecar): `cloudflared` egresses to the edge on port 7844 and
reached cluster Services from `http://<service>.<namespace>.svc.cluster.local:port` in Cloudflare routes.

## Deployment Strategy

Deployed by Flux on the `oci` k3s cluster via a `HelmRelease` (`helmrelease.yaml`) that renders the
local chart in this folder. The tunnel token is injected from the `cloudflared-tunnel-token` Secret
(via `valuesFrom`) and consumed as `TUNNEL_TOKEN` by the pod.

## Prerequisites

Works alongside the same tunnel-stack pattern used on the docker host, but uses a **separate, parallel
tunnel** with its own token. The token secret is created out-of-band because Flux cannot create it —
see [the cluster runbook](../../README.md) Step 4:

```bash
kubectl create namespace cloudflared
kubectl -n cloudflared create secret generic cloudflared-tunnel-token --from-literal=token="$CLOUDFLARE_CLUSTER_TUNNEL_TOKEN"
```

Until the Secret exists the HelmRelease errors; Flux retries on each reconcile interval.

## Updates

- **Chart/templates**: change files here and merge to `main` — Flux re-renders and upgrades.
- **Image version**: Renovate watches `values.yaml` (`tag:` field) and opens PRs.

Routes are managed in the Cloudflare Zero Trust dashboard (Networking → Tunnels), not in this repo.

## Current public hostname routes

Applied to the cluster tunnel as Public Hostname entries in the dashboard. Path
matching follows cloudflared's top-down ingress rules; the full request path is
forwarded to the origin and non-matching paths fall through to the catch-all
(404), so only the listed paths are exposed:

| Hostname | Path | Service |
|---|---|---|
| `auth.lorenzobaronio.com` | `/realms/*` | `http://keycloak-service.keycloak.svc.cluster.local:8080` |
| `auth.lorenzobaronio.com` | `/resources/*` | `http://keycloak-service.keycloak.svc.cluster.local:8080` |

These expose Keycloak's OIDC + theme endpoints publicly; the admin console and
everything else stay tailnet-only (see `04.identity/README.md`).