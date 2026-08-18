# Home Lab

My home lab setup.

## Documentation

See individual component READMEs for detailed instructions and configuration.

## 01 - k3s cluster

A single-node k3s Kubernetes cluster with Tailscale networking, Longhorn persistent storage, and application deployments.

### Deployment Order

Deploy components in the following order to ensure all dependencies are met:

1. **Cluster Setup**: Follow the bootstrap steps in [01.k3s-cluster/README.md](01.k3s-cluster/README.md)
2. **Networking** ([01.k3s-cluster/01.networking/](01.k3s-cluster/01.networking/)):
   - Deploy Tailscale Operator first (all other components depend on this)
3. **Storage** ([01.k3s-cluster/02.storage/](01.k3s-cluster/02.storage/)):
   - Deploy Longhorn after networking is ready
4. **Observability** ([01.k3s-cluster/03.observability/](01.k3s-cluster/03.observability/)):
   - Deploy Grafana Cloud observability after networking and storage

### Key Components

#### Tailscale Operator (Networking)

Provides secure ingress and networking via Tailscale proxy groups. All cluster components use this for network access.

**Location**: [01.k3s-cluster/01.networking/tailscale-operator](01.k3s-cluster/01.networking/tailscale-operator)

#### Longhorn (Storage)

Distributed block storage system for persistent volumes and automated backups.

**Location**: [01.k3s-cluster/02.storage/longhorn](01.k3s-cluster/02.storage/longhorn)

#### Grafana Cloud Observability

Cloud-first observability stack for metrics, logs, traces, and alerting with free-tier safeguards.

**Location**: [01.k3s-cluster/03.observability/grafana-cloud](01.k3s-cluster/03.observability/grafana-cloud)

## 02 - Docker Host

A dedicated Docker host running networking components (Cloudflare Tunnel + Tailscale) for standalone containerized services.

### Deployment Order

Deploy components in the following order:

1. **Networking** ([02.docker-host/01.networking/](02.docker-host/01.networking/)):
   - Deploy Cloudflare Tunnel + Tailscale first (all other services depend on this).

## 03 - Second k3s cluster

A second single-node k3s Kubernetes cluster (`oci`), isolated from `01.k3s-cluster`. Hosted on an Oracle Cloud Infrastructure VM (`luka`) and reached over Tailscale.

### Deployment Order

Deploy components in the following order to ensure all dependencies are met:

1. **Cluster Setup**: Follow the bootstrap steps in [03.k3s-cluster/README.md](03.k3s-cluster/README.md)
2. **Networking** ([03.k3s-cluster/01.networking/](03.k3s-cluster/01.networking/)):
   - Deploy Tailscale Operator first (all other components depend on this)
3. **Applications** ([03.k3s-cluster/99.apps/](03.k3s-cluster/99.apps/)):
   - Deploy the Homepage dashboard after networking is ready

### Key Components

#### Tailscale Operator (Networking)

Provides secure ingress and networking via Tailscale proxy groups. The operator registers as `tailscale-operator-oci` on the tailnet so it does not clash with the `01.k3s-cluster` operator.

**Location**: [03.k3s-cluster/01.networking/tailscale-operator](03.k3s-cluster/01.networking/tailscale-operator)

#### Homepage (Application)

Self-hosted application dashboard accessing the homelab over the tailnet.

**Location**: [03.k3s-cluster/99.apps/homepage](03.k3s-cluster/99.apps/homepage)

## Automatic Updates

### Dependabot

Watches Helm chart dependencies and opens PRs when updates are available.

Configuration: [.github/dependabot.yml](.github/dependabot.yml)

### Renovate

Scans for dependency updates and image version changes.

Configuration: [.github/renovate.json](.github/renovate.json)

### GitHub Workflows

Automated deployment workflows apply changes to the k3s cluster when merged to `main`.

- [tailscale-operator-upgrade.yml](.github/workflows/tailscale-operator-upgrade.yml)
- [tailscale-operator-oci-upgrade.yml](.github/workflows/tailscale-operator-oci-upgrade.yml)
- [longhorn-upgrade.yml](.github/workflows/longhorn-upgrade.yml)
- [grafana-cloud-observability-upgrade.yml](.github/workflows/grafana-cloud-observability-upgrade.yml)
- [homepage-oci-upgrade.yml](.github/workflows/homepage-oci-upgrade.yml)
- [node-os-maintenance.yml](.github/workflows/node-os-maintenance.yml)
