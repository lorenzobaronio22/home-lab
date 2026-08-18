## Kubernetes Operator

[Docs references](https://tailscale.com/docs/features/kubernetes-operator)

## Files in this folder

- `Chart.yaml`: pins the Tailscale operator chart dependency version (watched by Dependabot).
- `values.yaml`: non-secret operator values. The operator registers its API server proxy node as `tailscale-operator-oci` on the tailnet.
- `tailscale.yaml`: optional ProxyGroup manifest for ingress high availability.
- `nginx-hello.yaml`: optional example service exposure.

## Automatic updates with Dependabot

Dependabot is configured in `.github/dependabot.yml` to watch `03.k3s-cluster/01.networking/tailscale-operator/Chart.yaml`.

When a new `tailscale-operator` chart version is available, Dependabot opens a PR that bumps the version in `Chart.yaml`.

## Automatic apply to the `oci` cluster after merge

Workflow: `.github/workflows/tailscale-operator-oci-upgrade.yml`

Trigger:

- Push to `main` with changes under `03.k3s-cluster/01.networking/tailscale-operator/**`
- Manual run (`workflow_dispatch`)

The workflow:
1. Uses a GitHub-hosted runner and joins your tailnet through `tailscale/github-action` with `tag:ci` identity.
2. Configures kubeconfig via `tailscale configure kubeconfig tailscale-operator-oci` (the operator's API server proxy node).
3. Runs `helm upgrade --install` against the `oci` cluster with the scoped `github-runner` identity (cannot escalate privileges or access system namespaces).
4. Helm dependency updates and applies the tailscale-operator release to the `tailscale` namespace.

The runner identity is restricted to Helm deployment operations and cannot modify cluster RBAC or access system namespaces.

### Required GitHub repository secrets

- `TS_OAUTH_CLIENT_ID`: Tailscale OAuth credentials client-id bound to CI tag identity to create ephemeral reusable auth keys.
- `TS_OAUTH_SECRET`: Tailscale OAuth credentials client-secret bound to CI tag identity to create ephemeral reusable auth keys.
- `TS_OPERATOR_OAUTH_CLIENT_ID`: Tailscale operator OAuth client ID.
- `TS_OPERATOR_OAUTH_CLIENT_SECRET`: Tailscale operator OAuth client secret.

## First-time installation (manual)

The upgrade workflow reaches the cluster through the operator's API server proxy, so the operator must already be running before the workflow can be used. Install it once manually by pointing Helm at the `oci` cluster's kubeconfig.

The operator is reached over the tailnet as `luka` (MagicDNS), which is also the API server `tls-san` configured at bootstrap.

```bash
scp root@luka:/etc/rancher/k3s/k3s.yaml ~/.kube/luka.yaml
sed -i '' 's|server: https://127.0.0.1:6443|server: https://luka:6443|' ~/.kube/luka.yaml

helm dependency update 03.k3s-cluster/01.networking/tailscale-operator

export TS_OPERATOR_OAUTH_CLIENT_ID="<operator-oauth-client-id>"
export TS_OPERATOR_OAUTH_CLIENT_SECRET="<operator-oauth-client-secret>"

helm upgrade --install tailscale-operator 03.k3s-cluster/01.networking/tailscale-operator \
  --namespace tailscale --create-namespace \
  --values 03.k3s-cluster/01.networking/tailscale-operator/values.yaml \
  --set-string tailscale-operator.oauth.clientId="$TS_OPERATOR_OAUTH_CLIENT_ID" \
  --set-string tailscale-operator.oauth.clientSecret="$TS_OPERATOR_OAUTH_CLIENT_SECRET" \
  --kubeconfig ~/.kube/luka.yaml \
  --wait
```

Verify the operator node `tailscale-operator-oci` appears in the Tailscale admin console and that the tailnet ACL allows the `tag:ci` identity to reach it.

## Setup ProxyGroup for ingress HA

```bash
export KUBECONFIG="/Users/lorenzobaronio/.kube/luka.yaml"
kubectl -n tailscale apply -f 03.k3s-cluster/01.networking/tailscale-operator/tailscale.yaml
```

## [Optional] Expose a NGINX example service

[Documentation Reference](https://tailscale.com/docs/features/kubernetes-operator/how-to/cluster-ingress)

```bash
kubectl apply -f 03.k3s-cluster/01.networking/tailscale-operator/nginx-hello.yaml
kubectl delete -f 03.k3s-cluster/01.networking/tailscale-operator/nginx-hello.yaml
```
