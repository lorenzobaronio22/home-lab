# Plan: 3-Node HA K3s Cluster with Embedded etcd

## Summary
Create a high-availability K3s cluster with 3 nodes running both server (control-plane + etcd) and agent roles using embedded etcd for distributed storage.

## Node Configuration
- **Node 1**: 100.102.112.61 (first server - cluster-init)
- **Node 2**: 100.75.90.119 (join as server)
- **Node 3**: 100.68.137.96 (join as server)

## Installation Steps

### Step 1: Initialize First Server (Node 1)
SSH to 100.102.112.61 and run:
```bash
curl -sfL https://get.k3s.io | K3S_TOKEN=mysecretpassword sh -s - server \
  --cluster-init \
  --tls-san=100.102.112.61
```

### Step 2: Get Node Token from First Server
On Node 1, retrieve the token:
```bash
cat /var/lib/rancher/k3s/server/node-token
```

### Step 3: Join Second Server (Node 2)
SSH to 100.75.90.119 and run (replace TOKEN with value from Step 2):
```bash
curl -sfL https://get.k3s.io | K3S_TOKEN=TOKEN sh -s - server \
  --server https://100.102.112.61:6443 \
  --tls-san=100.75.90.119
```

### Step 4: Join Third Server (Node 3)
SSH to 100.68.137.96 and run (replace TOKEN with value from Step 2):
```bash
curl -sfL https://get.k3s.io | K3S_TOKEN=TOKEN sh -s - server \
  --server https://100.102.112.61:6443 \
  --tls-san=100.68.137.96
```

### Step 5: Verify Cluster
On any server node, check cluster status:
```bash
kubectl get nodes
```

### Step 6: Configure Local kubectl
On your local machine:
```bash
# Copy kubeconfig from any server node
scp root@100.102.112.61:/etc/rancher/k3s/k3s.yaml ~/.kube/config

# Edit the server URL (replace localhost IP with any server IP)
# Change: server: https://127.0.0.1:6443
# To: server: https://100.102.112.61:6443
```

## Verification
1. Run `kubectl get nodes` - should show 3 nodes with roles: control-plane,etcd,master
2. Run `kubectl get pods -A` - should show all system pods running
