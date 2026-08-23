# Home Lab

My home lab setup.

## Documentation

See individual component READMEs for detailed instructions and configuration.

## 02 - Docker Host

A dedicated Docker host running networking components (Cloudflare Tunnel + Tailscale) for standalone containerized services.

### Deployment Order

Deploy components in the following order:

1. **Networking** ([02.docker-host/01.networking/](02.docker-host/01.networking/)):
   - Deploy Cloudflare Tunnel + Tailscale first (all other services depend on this).

## 03 - k3s cluster (oci)

A single-node k3s Kubernetes cluster (`oci`), hosted on an Oracle Cloud Infrastructure VM (`luka`) and reached over Tailscale.

### Deployment Order

1. **Cluster Setup**: Follow the bootstrap steps in [03.k3s-cluster/README.md](03.k3s-cluster/README.md)
2. **Everything else**: Flux reconciles networking first, then applications (`dependsOn` ordering)

### Key Components

#### Tailscale Operator (Networking)

Provides secure ingress and networking via Tailscale proxy groups. The operator registers as `tailscale-operator-oci` on the tailnet.

**Location**: [03.k3s-cluster/01.networking/tailscale-operator](03.k3s-cluster/01.networking/tailscale-operator)

#### Homepage (Application)

Self-hosted application dashboard accessing the homelab over the tailnet.

**Location**: [03.k3s-cluster/99.apps/homepage](03.k3s-cluster/99.apps/homepage)

## Automatic Updates

### GitOps (Flux)

The k3s cluster is reconciled by [Flux](https://fluxcd.io) from this repository. Merging to `main`
is the only action needed to change the cluster — see the
[cluster runbook](03.k3s-cluster/README.md) for bootstrap, rebuild and update semantics.

### Dependabot

Watches GitHub Actions and docker-compose dependencies and opens PRs when updates are available.

Configuration: [.github/dependabot.yml](.github/dependabot.yml)

### Renovate

Scans for dependency updates: Flux `HelmRelease` chart versions and container image tags.

Configuration: [.github/renovate.json](.github/renovate.json)

### GitHub Workflows

- [cloudflare-tunnel-upgrade.yml](.github/workflows/cloudflare-tunnel-upgrade.yml) (docker host)
- [renovate.yml](.github/workflows/renovate.yml)
