# Networking Components

This folder contains networking infrastructure components for the Docker host.

## Components

- **cloudflared**: Cloudflare Tunnel agent, sharing the Tailscale node's network namespace.
- **tailscale**: Tailscale node providing the host's tailnet identity and WireGuard transport.

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
