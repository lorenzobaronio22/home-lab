# Plan: 3-Node HA K3s Cluster with Embedded etcd

## Summary
Create a high-availability K3s cluster with 3 nodes running server role (control-plane + etcd) using embedded etcd.

## Node Configuration
- `k3s1`: first server (`--cluster-init`)
- `k3s2`: join as server
- `k3s3`: join as server

## Bootstrap Steps (one-time, manual)

Set your own private addresses and token in your shell first:

```bash
export K3S1_ADDR="<private-node-1-address>"
export K3S2_ADDR="<private-node-2-address>"
export K3S3_ADDR="<private-node-3-address>"
export K3S_TOKEN="<strong-shared-secret>"
```

### Step 1: Initialize First Server (`k3s1`)

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN="$K3S_TOKEN" sh -s - server \
  --cluster-init \
  --tls-san="$K3S1_ADDR"
```

### Step 2: Join Second Server (`k3s2`)

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN="$K3S_TOKEN" sh -s - server \
  --server "https://$K3S1_ADDR:6443" \
  --tls-san="$K3S2_ADDR"
```

### Step 3: Join Third Server (`k3s3`)

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN="$K3S_TOKEN" sh -s - server \
  --server "https://$K3S1_ADDR:6443" \
  --tls-san="$K3S3_ADDR"
```

### Step 4: Verify Cluster

```bash
kubectl get nodes
kubectl get pods -A
```

### Step 5: Configure Local kubectl

```bash
# Copy kubeconfig from first server
scp "root@$K3S1_ADDR:/etc/rancher/k3s/k3s.yaml" ~/.kube/config

# Edit server URL in kubeconfig
# Change: server: https://127.0.0.1:6443
# To:     server: https://<your-private-k3s-server-address>:6443
```

## Day-2 OS Maintenance Automation

Workflow: `.github/workflows/node-os-maintenance.yml`

What it does:
1. Runs on a schedule (plus manual `workflow_dispatch`).
2. Selects one node per run using ISO week rotation (`week % 3`): `k3s1`, then `k3s2`, then `k3s3`.
3. Validates cluster health prechecks.
4. Cordon target node.
5. Applies Ubuntu 24.04 updates (`apt-get update`, `dist-upgrade`, `autoremove`).
6. Reboots target node.
7. Waits for node to come back, verifies `k3s` service and Kubernetes `Ready` status.
8. Uncordons target node only after successful post-check.
9. Sends Telegram success/failure alerts.

This design updates one node per week so the other two nodes stay available if one update fails.

### GitHub Repository Variables

- `TS_IP_K3S1`: Tailscale IP for `k3s1`
- `TS_IP_K3S2`: Tailscale IP for `k3s2`
- `TS_IP_K3S3`: Tailscale IP for `k3s3`
- `MAINTENANCE_PAUSED`: set to `true` to skip scheduled maintenance

### GitHub Repository Secrets

- `TS_OAUTH_CLIENT_ID`: tailscale OAuth credential to join github-runner to tailnet
- `TS_OAUTH_SECRET`: tailscale OAuth credential to join github-runner to tailnet
- `TELEGRAM_BOT_TOKEN`: telegram bot credential for chat alert
- `TELEGRAM_CHAT_ID`: telegram bot chat identifier for alert
