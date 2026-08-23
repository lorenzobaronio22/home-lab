# Applications

This folder contains application manifests and Helm charts deployed to the `oci` k3s cluster.

Each subfolder represents a standalone application with its own Helm chart and deployment configuration, reconciled by Flux (see `03.k3s-cluster/clusters/oci/apps-kustomization.yaml`).

## Prerequisites

Applications require networking setup (see `03.k3s-cluster/01.networking/`). The ordering is enforced by Flux via `dependsOn` on the cluster-level Kustomizations.