# Applications

This folder contains all application manifests and Helm charts deployed to the k3s cluster.

Each subfolder represents a standalone application with its own Helm chart and deployment configuration.

## Prerequisites

All applications require:
- Networking setup (see `01.k3s-cluster/01.networking/`)
- Storage setup (see `01.k3s-cluster/02.storage/`)
