# Cloudflare Tunnel Agent

Remotely-managed Cloudflare Tunnel connecting the `oci` k3s cluster to Cloudflare's edge, following
Cloudflare's [Kubernetes deployment guide](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/deployment-guides/kubernetes/).
Runs as a standalone pod (no Tailscale sidecar): `cloudflared` egresses to the edge on port 7844 and
reached cluster Services from `http://<service>.<namespace>.svc.cluster.local:port` in Cloudflare routes.

## Tunnel topology (do not share tokens)

This tunnel is **dedicated to the k3s cluster** and must use its **own token**, distinct from the
docker-host tunnel (`02.docker-host/01.networking`). Running the same token on both hosts makes
Cloudflare treat the two cloudflared containers as redundant connectors of ONE tunnel and
load-balances requests across them; a connector cannot reach an origin it has no route to
(cluster-only service DNS vs docker/tailnet names), which shows up as random `502` on any
connector that can't resolve the origin. If that happens, migrate to separate tunnels as described
in the [migration checklist](#migration-checklist) below.

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
This cluster tunnel routes the Keycloak public endpoints and the WordPress site below; the remaining
hostnames (`im-learning.app`, `notes.*`, `vault.*`, `media.*`) live on the **docker-host tunnel**,
whose connector can reach docker containers and the tailnet but not `*.svc.cluster.local`. The
`gemgarden.org` hostnames moved from the docker-host tunnel to this one during the WordPress
migration (split-tunnel rule: never put the same hostname on both tunnels).

## Current public hostname routes

Applied to the cluster tunnel as Public Hostname entries in the dashboard. Path
matching follows cloudflared's top-down ingress rules; the full request path is
forwarded to the origin and non-matching paths fall through to the catch-all
(404), so only the listed paths are exposed:

| Hostname | Path | Service |
|---|---|---|
| `auth.lorenzobaronio.com` | `/realms/*` | `http://keycloak-service.keycloak.svc.cluster.local:8080` |
| `auth.lorenzobaronio.com` | `/resources/*` | `http://keycloak-service.keycloak.svc.cluster.local:8080` |
| `gemgarden.org` | `/` | `http://gemgarden-wordpress.gemgarden-wordpress.svc.cluster.local` |
| `www.gemgarden.org` | `/` | `http://gemgarden-wordpress.gemgarden-wordpress.svc.cluster.local` |

These expose Keycloak's OIDC + theme endpoints and the WordPress site publicly;
the admin console and everything else stay tailnet-only (see
`04.identity/README.md` / `06.gemgarden-wordpress/README.md`).

## Migration checklist

If the cluster and docker-host cloudflared ever shared the same token (same tunnel), split them
back into two tunnels via the Zero Trust dashboard:

1. Create a **new tunnel** (e.g. `oci-k3s`) and copy its token — this is now `CLOUDFLARE_CLUSTER_TUNNEL_TOKEN`.
2. On the new tunnel add the two `auth.lorenzobaronio.com` Public Hostname routes from the table
   above (the dashboard replaces the `auth` CNAME with the new tunnel's cfargotunnel.com target).
3. Point the cluster at the new tunnel:
   ```bash
   kubectl -n cloudflared delete secret cloudflared-tunnel-token
   kubectl -n cloudflared create secret generic cloudflared-tunnel-token \
     --from-literal=token="$CLOUDFLARE_CLUSTER_TUNNEL_TOKEN"
   kubectl -n cloudflared rollout restart deployment cloudflared
   ```
4. From the **old (docker-host) tunnel**, remove the `auth.lorenzobaronio.com` routes so it no
   longer claims that hostname (keep `gemgarden`, `im-learning`, `notes`, `vault`, `media` there —
   those origins resolve from the docker host/tailnet, never from the cluster).
5. Verify:
   ```bash
   curl -i https://auth.lorenzobaronio.com/realms/master/.well-known/openid-configuration
   curl -i https://auth.lorenzobaronio.com/admin   # expect 404 (fall-through)
   kubectl -n cloudflared logs deploy/cloudflared | rg 'auth.lorenzobaronio'   # 502s gone
   ```