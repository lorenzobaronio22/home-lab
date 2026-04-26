## Kubernetes Operator

[Docs references](https://tailscale.com/docs/features/kubernetes-operator)

## Files in this folder

- `Chart.yaml`: pins the Tailscale operator chart dependency version (watched by Dependabot).
- `values.yaml`: non-secret operator values.
- `tailscale.yaml`: optional ProxyGroup manifest for ingress high availability.
- `nginx-hello.yaml`: optional example service exposure.

## Install or upgrade locally (manual)

```bash
helm repo add tailscale https://pkgs.tailscale.com/helmcharts
helm repo update
helm dependency update ./02.tailscale-operator
export CLIENT_ID=<OAuth client ID>
export CLIENT_SECRET=<OAuth client secret>
helm upgrade \
  --install \
  tailscale-operator \
  ./02.tailscale-operator \
  --namespace=tailscale \
  --create-namespace \
  --values ./02.tailscale-operator/values.yaml \
  --set-string tailscale-operator.oauth.clientId="$CLIENT_ID" \
  --set-string tailscale-operator.oauth.clientSecret="$CLIENT_SECRET" \
  --wait
```

## Automatic updates with Dependabot

Dependabot is configured in `.github/dependabot.yml` to watch `02.tailscale-operator/Chart.yaml`.

When a new `tailscale-operator` chart version is available, Dependabot opens a PR that bumps the version in `Chart.yaml`.

## Automatic apply to k3s after merge

Workflow: `.github/workflows/tailscale-operator-upgrade.yml`

Trigger:

- Push to `main` with changes under `02.tailscale-operator/**`
- Manual run (`workflow_dispatch`)

The workflow:
1. Uses a GitHub-hosted runner and joins your tailnet through `tailscale/github-action` with `tag:ci` identity.
2. Loads kubeconfig from the `KUBECONFIG_B64` secret (generated from the `github-runner` service account).
3. Runs `helm upgrade --install` against your cluster with the scoped `github-runner` identity (cannot escalate privileges or access system namespaces).
4. Helm dependency updates and applies the tailscale-operator release to the `tailscale` namespace.

The runner identity is restricted to Helm deployment operations and cannot modify cluster RBAC or access system namespaces.

### Required GitHub repository secrets

- `TS_OAUTH_CLIENT_ID`: Tailscale OAuth credentials client-id bound to CI tag identity to create ephemeral reusable auth keys.
- `TS_OAUTH_SECRET`: Tailscale OAuth credentials client-secret bound to CI tag identity to create ephemeral reusable auth keys.
- `KUBECONFIG_B64`: Base64-encoded kubeconfig for the `github-runner` service account (generated via `01.k3s-cluster-init/rbac/github-runner-sa.yaml`). See [Step 8 in 01.k3s-cluster-init/README.md](../01.k3s-cluster-init/README.md#step-8-generate-kubeconfig-for-github-actions) for generation steps.
- `TS_OPERATOR_OAUTH_CLIENT_ID`: Tailscale operator OAuth client ID.
- `TS_OPERATOR_OAUTH_CLIENT_SECRET`: Tailscale operator OAuth client secret.

### Kubernetes RBAC prerequisites

The workflow uses a dedicated, scoped service account `github-runner` created in the `github-runners` namespace. This account has limited permissions:
- ✅ Can deploy to and manage resources in user-created namespaces.
- ✅ Can create new namespaces.
- ❌ Cannot access system namespaces (kube-system, kube-public, kube-node-lease).
- ❌ Cannot modify RBAC or cluster-scoped resources.

See [01.k3s-cluster-init/rbac/github-runner-sa.yaml](../01.k3s-cluster-init/rbac/github-runner-sa.yaml) for full RBAC definitions.

### Tailnet policy prerequisites

- Create a tag for CI nodes (for example `tag:ci`).
- Ensure the auth key is tag-scoped, reusable, and ephemeral.
- Allow `tag:ci` to reach the Kubernetes API endpoint used by `KUBECONFIG_B64`.

## Setup ProxyGroup for ingress HA

```bash
kubectl -n tailscale apply -f tailscale.yaml
```

## [Optional] Expose a NGINX example service

[Documentation Reference](https://tailscale.com/docs/features/kubernetes-operator/how-to/cluster-ingress)

```bash
kubectl apply -f nginx-hello.yaml
```
