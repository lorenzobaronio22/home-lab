# Networking Components

This folder contains networking infrastructure components deployed to the `oci` k3s cluster.

## Components

- **tailscale-operator**: Secure ingress and networking via Tailscale proxy groups. Deployed by Flux
  as a `HelmRelease` pinned to the upstream chart (`https://pkgs.tailscale.com/helmcharts`).

## Deployment Order

Flux enforces this automatically: the cluster-level `apps` Kustomization declares
`dependsOn: networking`, so applications are only deployed once networking is healthy.
