# Homepage Application

A self-hosted application dashboard providing a centralized view of services and applications in the homelab, reached at `homepage.tail10187.ts.net`.

## Deployment Strategy

Deployed by Flux on the `oci` k3s cluster via a `HelmRelease` (`helmrelease.yaml`) that renders the
local chart in this folder. The ingress uses the `tailscale` class through the
`private-ingress-proxies` ProxyGroup and serves on the `homepage` tailnet hostname.

## Updates

- **Chart/templates**: change files here and merge to `main` — Flux re-renders and upgrades.
- **Image version**: Renovate watches `values.yaml` (`tag:` field) and opens PRs.

There is no deploy workflow anymore; see the [cluster runbook](../../README.md) for how
reconciliation works.
