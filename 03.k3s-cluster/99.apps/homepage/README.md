# Homepage Application

A self-hosted application dashboard providing a centralized view of services and applications in the homelab, reached at `homepage.tail10187.ts.net`.

## Deployment Strategy

Deployed as a Helm chart on the `oci` k3s cluster. The ingress uses the `tailscale` class through the `private-ingress-proxies` ProxyGroup, so it serves on the same `homepage` tailnet hostname previously used from `01.k3s-cluster`.

## Installation

See the workflow at `.github/workflows/homepage-oci-upgrade.yml` for automated deployment.