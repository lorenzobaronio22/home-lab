# Plan: Single-Node K3s Cluster (`oci`) — Flux GitOps

## Summary
A single-node K3s cluster, hosted on an Oracle Cloud Infrastructure VM and reached over Tailscale.
All workloads are deployed and kept in sync by [Flux v2](https://fluxcd.io): this repository is the single
source of truth. Merging to `main` is the only action ever required to change the cluster.

```
git push/merge ──► GitHub ──pull──► Flux (in cluster) ──reconcile──► k3s
```

## Node Configuration
- `luka`: single server (`--cluster-init`), Tailscale SSH as `root@luka`
- Assumptions: Ubuntu VM on OCI, root login via Tailscale SSH enabled, MagicDNS resolves `luka`

## Rebuild Runbook (from scratch)

Only steps 1–6 are manual. Everything else is reconciled automatically from Git.

### Step 1: Initialize Server (`luka`)

Set the address and token in your shell first. `LUKA_ADDR` must match what you will use to reach the API server (Tailscale hostname `luka`, full MagicDNS name, or the tailnet IP):

```bash
export LUKA_ADDR="luka"
export K3S_TOKEN="<strong-shared-secret>"
ssh root@luka
curl -sfL https://get.k3s.io | K3S_TOKEN="$K3S_TOKEN" sh -s - server \
  --cluster-init \
  --tls-san="$LUKA_ADDR"
exit
```

The token is for future agent/server joins; a single server would work without it.

### Step 2: Verify Cluster

```bash
ssh root@luka "kubectl get nodes && kubectl get pods -A"
```

Expect `luka` as a `Ready` node and the control-plane pods running.

### Step 3: Temporary API Access from Your Laptop

`flux bootstrap` only needs HTTPS access to the Kubernetes API (port 6443). Pick one:

- **SSH tunnel** (simplest, nothing extra installed on the VM):
  ```bash
  ssh -L 6443:localhost:6443 root@luka -N   # keep running in a terminal
  scp root@luka:/etc/rancher/k3s/k3s.yaml /tmp/luka-kubeconfig
  # the kubeconfig already points at 127.0.0.1 — leave it as is; the tunnel maps it to the VM
  export KUBECONFIG=/tmp/luka-kubeconfig
  ```
  The kubeconfig references `127.0.0.1`, which the tunnel maps to the VM's local API.

- **Tailscale on the host** (persistent alternative): install `tailscaled` on the VM itself
  (outside k3s — Flux cannot manage it) and reach `https://$LUKA_ADDR:6443` over the tailnet.
  Regenerate the kubeconfig with `--tls-san` already set in Step 1.

Verify: `kubectl --kubeconfig=/tmp/luka-kubeconfig get nodes`

### Step 4: Create Prerequisites Flux Cannot Provide

Flux deploys everything in this repo, but two things live outside its reach:

```bash
# Tailscale Operator OAuth credentials (used by HelmRelease via valuesFrom)
kubectl create namespace tailscale
kubectl -n tailscale create secret generic tailscale-operator-oauth \
  --from-literal=client-id="$TS_OPERATOR_OAUTH_CLIENT_ID" \
  --from-literal=client-secret="$TS_OPERATOR_OAUTH_CLIENT_SECRET"
```

### Step 5: Bootstrap Flux

Requires the `flux` CLI locally ([install](https://fluxcd.io/flux/installation/)) and a GitHub PAT with
`repo` + `workflow` scopes:

```bash
flux bootstrap github \
  --owner=lorenzobaronio22 \
  --repository=home-lab \
  --branch=main \
  --path=03.k3s-cluster/clusters/oci \
  --personal
```

This one-time command:
1. Installs the Flux controllers into the `flux-system` namespace via the API
2. Commits those manifests to `03.k3s-cluster/clusters/oci/flux-system/`
3. Registers a **read-only deploy key** so the cluster can pull this repo

Afterwards the CLI is no longer needed for day-to-day operation: the `source-controller` pod inside
the cluster clones the repo on its own schedule forever. The machine that ran bootstrap can be deleted
from all configuration afterwards.

### Step 6: Watch Reconciliation

```bash
flux get kustomizations --watch     # networking → apps, in order
flux get helmreleases -A            # tailscale-operator, homepage
kubectl get pods -A
```

First reconciliation takes a few minutes: Flux applies networking first (the `apps` Kustomization has
`dependsOn: networking`), then applications. The Tailscale Operator's HelmRelease will error until the
OAuth Secret from Step 4 exists — Flux retries on each reconcile interval, no action needed beyond fixing the cause.

### Step 7: Clean Up Temporary Access

Close the SSH tunnel and delete `/tmp/luka-kubeconfig`. Day-to-day access goes back to being
read-only through the tailnet; changes happen exclusively through Git.

## How Updates Work

| Action | What happens |
|---|---|
| Merge PR changing values/manifests | Cluster converges within seconds–minutes |
| Renovate PR bumps chart/image version | Merge → helm-controller upgrades the release |
| Manual `kubectl` change in cluster | Reverted on next reconcile ("drift" self-healing) |
| Rollback | `git revert <commit>` — Git history is the audit log |
| Upgrade Flux itself | Renovate bumps components in `clusters/oci/flux-system/gotk-components.yaml` → Flux re-applies itself |

Renovate parses the `HelmRelease` manifests directly (see `.github/renovate.json`), so chart version
bumps arrive as ordinary PRs against the pinned `version:` fields.

## Repository Layout

```
03.k3s-cluster/
├── clusters/oci/
│   ├── flux-system/                    # Flux controllers (committed by bootstrap)
│   ├── networking-kustomization.yaml   # → ../../01.networking
│   └── apps-kustomization.yaml         # → ../../99.apps (dependsOn networking)
├── 01.networking/
│   ├── kustomization.yaml
│   └── tailscale-operator/             # HelmRepository + HelmRelease + ProxyGroup
└── 99.apps/
    ├── kustomization.yaml
    └── homepage/                       # local chart + HelmRelease
```

Note: this intentionally keeps manifests near their component folders rather than adopting Flux's
canonical `apps/`+`infrastructure/` layout — fine at homelab scale.

## Networking Components

See [01.networking/README.md](01.networking/README.md). The Tailscale Operator provides the `tailscale`
ingress class used by application manifests; its API-server proxy (`tailscale-operator-oci`) remains
useful for read-only inspection from CI or your laptop.
