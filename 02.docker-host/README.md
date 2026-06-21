# Docker Host

Dedicated host for standalone Docker containers and networking components.

## Node Configuration
- Single Docker host (Ubuntu)

## Deployment Order

Deploy components in the following order:

1. **Networking** ([02.docker-host/01.networking/](02.docker-host/01.networking/)):
   - Deploy Cloudflare Tunnel + Tailscale first (all other services depend on this).
