# Plan: Second Single-Node K3s Cluster (`oci`)

## Summary
Second single-node K3s cluster, isolated from `01.k3s-cluster`. A single server running the control-plane and embedded etcd, hosted on an Oracle Cloud Infrastructure VM and reached over Tailscale.

## Node Configuration
- `luka`: single server (`--cluster-init`), Tailscale SSH as `root@luka`
- Assumptions: Ubuntu VM on OCI, root login via Tailscale SSH enabled, MagicDNS resolves `luka`

## Bootstrap Steps (one-time, manual)

Set the address and token in your shell first. `LUKA_ADDR` must match what you will use to reach the API server (Tailscale hostname `luka`, full MagicDNS name, or the tailnet IP):

```bash
export LUKA_ADDR="luka"
export K3S_TOKEN="<strong-shared-secret>"
```

### Step 1: Initialize Server (`luka`)

```bash
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

## Networking

Once the cluster is up, deploy the Tailscale Operator before any other components. It provides the `tailscale` ingress class used by storage and application manifests, and its API server proxy (`tailscale-operator-oci`) is what the automatic-update workflow uses to reach this cluster.

See [01.networking/README.md](01.networking/README.md) for details and the manual first-time install steps.
