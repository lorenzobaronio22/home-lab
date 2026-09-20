# Networking Components

This folder contains networking infrastructure components deployed to the `oci` k3s cluster.

## Components

- **tailscale-operator**: Secure ingress and networking via Tailscale proxy groups. Deployed by Flux
  as a `HelmRelease` pinned to the upstream chart (`https://pkgs.tailscale.com/helmcharts`).
- **cloudflared**: Remotely-managed Cloudflare Tunnel agent exposing cluster services to the public
  Internet. Deployed by Flux as a `HelmRelease` rendering the local chart in
  [`cloudflared/`](cloudflared/); requires the `cloudflared-tunnel-token` Secret from the runbook.

## Deployment Order

Flux enforces this automatically: the cluster-level `apps` Kustomization declares
`dependsOn: networking`, so applications are only deployed once networking is healthy.
