# Storage Infrastructure

This folder contains storage infrastructure components deployed to the k3s cluster.

## Components

- **longhorn**: Distributed block storage for persistent volumes and backup.

## Prerequisites

Before deploying storage components, ensure that:
- Networking (Tailscale Operator) is deployed first, as Longhorn's ingress depends on the Tailscale ingress class.
- All nodes have required packages: `open-iscsi` and `nfs-common`.
