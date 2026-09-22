# Networking Components

This folder contains networking infrastructure components for the Docker host.

## Components

- **cloudflared**: Cloudflare Tunnel agent, sharing the Tailscale node's network namespace.
- **tailscale**: Tailscale node providing the host's tailnet identity and WireGuard transport.

The tunnel token (`CLOUDFLARE_TOKEN`) must be **distinct** from the k3s cluster tunnel token.
Sharing the same token registers both cloudflared instances as connectors of one tunnel, and
Cloudflare load-balances requests across them — a connector picked for an origin it can't resolve
(cluster-only `*.svc.cluster.local`, or docker/tailnet-only names) answers `502`. Each tunnel keeps
only the origins its connector can actually reach:

- **Docker-host tunnel (this file)** — docker-container and tailnet origins: `im-learning.app`
  (+`www`, `lab`), `notes.lorenzobaronio.com` (docmost), `vault.lorenzobaronio.com`
  (vaultwarden), `media.lorenzobaronio.com` (jellyfin on the tailnet).
  `gemgarden.org` (+`www`) moved to the cluster tunnel when the WordPress site
  was migrated into k3s (`03.k3s-cluster/01.networking/cloudflared/README.md`).
- **Cluster tunnel (`03.k3s-cluster/01.networking/cloudflared`)** — Keycloak
  (`auth.lorenzobaronio.com` `/realms/*` + `/resources/*`) and the WordPress
  site (`gemgarden.org`, `www.gemgarden.org` → the k3s Service).

## Deployment Order

Deploy Cloudflare Tunnel and Tailscale before any other services on this host.

## Prerequisites

- Ubuntu-based Docker host.
- Cloudflare account with a Zero Trust tunnel configured.
- Tailscale account with a machine auth key (with `--tags=tag:container`).

## Docker Compose

The services are defined in `docker-compose.yml`. Key details:

- **cloudflared** shares the network namespace of the tailscale container via `network_mode: service:tailscale-cloudflare-tunnel`.
- The Tailscale container runs with device and capability access (`/dev/net/tun`, `net_admin`, `sys_module`) required for WireGuard.
- Tailscale state is persisted in `./tailscale/state` on the host filesystem.

### Version pins

- `cloudflare/cloudflared:2026.6.1`
- `tailscale/tailscale:v1.98.4`

### Environment variables

Defined in `.env.example`:

| Variable | Description |
|---|---|
| `CLOUDFLARE_TOKEN` | Tunnel token from Cloudflare Zero Trust dashboard |
| `TS_AUTHKEY` | Tailscale machine auth key (with `--tags=tag:container`) |

### Manual deployment

```bash
docker compose up -d
```

Verify with:

```bash
docker compose ps
docker compose logs cloudflared
docker compose logs tailscale-cloudflare-tunnel
```

The Tailscale node will appear in your admin console as `cloudflare-tunnel` (hostname) with the `tag:container` tag.
