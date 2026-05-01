# Networking Components

This folder contains networking infrastructure components deployed to the k3s cluster.

## Components

- **tailscale-operator**: Secure ingress and networking via Tailscale proxy groups.

## Deployment Order

Tailscale Operator should be deployed before any other cluster components, as it provides the ingress class (`tailscale`) used by storage and application manifests.
