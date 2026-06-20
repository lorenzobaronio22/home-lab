# Plan: Single-Node K3s Cluster

## Summary
Create a single-node K3s cluster with one server running the control-plane and embedded etcd.

## Node Configuration
- `k3s1`: single server (`--cluster-init`)

## Bootstrap Steps (one-time, manual)

Set your private address and token in your shell first:

```bash
export K3S1_ADDR="<private-node-1-address>"
export K3S_TOKEN="<strong-shared-secret>"
```

### Step 1: Initialize First Server (`k3s1`)

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN="$K3S_TOKEN" sh -s - server \
  --cluster-init \
  --tls-san="$K3S1_ADDR"
```

### Step 2: Verify Cluster

```bash
kubectl get nodes
kubectl get pods -A
```

### Step 3: Configure Local kubectl

```bash
# Copy kubeconfig from first server
scp "root@$K3S1_ADDR:/etc/rancher/k3s/k3s.yaml" ~/.kube/config

# Edit server URL in kubeconfig
# Change: server: https://127.0.0.1:6443
# To:     server: https://<your-private-k3s-server-address>:6443
```
