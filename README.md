# Home Lab

My home lab setup.

## Documentation

See individual component READMEs for detailed instructions
configuration

## 01 - k3s cluster

A high-availability k3s Kubernetes cluster with Tailscale networking, Longhorn distributed storage, and application deployments.

### Deployment Order

Deploy components in the following order to ensure all dependencies are met:

1. **Cluster Setup**: Follow the bootstrap steps in [01.k3s-cluster/README.md](01.k3s-cluster/README.md)
2. **Networking** ([01.k3s-cluster/01.networking/](01.k3s-cluster/01.networking/)):
   - Deploy Tailscale Operator first (all other components depend on this)
3. **Storage** ([01.k3s-cluster/02.storage/](01.k3s-cluster/02.storage/)):
   - Deploy Longhorn after networking is ready
4. **Applications** ([01.k3s-cluster/99.apps/](01.k3s-cluster/99.apps/)):
   - Deploy any applications after both networking and storage are ready

### Key Components

#### Tailscale Operator (Networking)

Provides secure ingress and networking via Tailscale proxy groups. All cluster components use this for network access.

**Location**: [01.k3s-cluster/01.networking/tailscale-operator](01.k3s-cluster/01.networking/tailscale-operator)

#### Longhorn (Storage)

Distributed block storage system for persistent volumes and automated backups.

**Location**: [01.k3s-cluster/02.storage/longhorn](01.k3s-cluster/02.storage/longhorn)

#### Applications

Deployed services and applications running on the cluster.

**Location**: [01.k3s-cluster/99.apps/](01.k3s-cluster/99.apps/)

### Automatic Updates

#### Dependabot

Watches Helm chart dependencies and opens PRs when updates are available.

Configuration: [.github/dependabot.yml](.github/dependabot.yml)

#### Renovate

Scans for dependency updates and image version changes.

Configuration: [.github/renovate.json](.github/renovate.json)

#### GitHub Workflows

Automated deployment workflows apply changes to the k3s cluster when merged to `main`.

- [tailscale-operator-upgrade.yml](.github/workflows/tailscale-operator-upgrade.yml)
- [longhorn-upgrade.yml](.github/workflows/longhorn-upgrade.yml)
- [homepage-upgrade.yml](.github/workflows/homepage-upgrade.yml)
- [node-os-maintenance.yml](.github/workflows/node-os-maintenance.yml)
