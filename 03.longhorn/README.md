# Longhorn Installation

[Reference documentation](https://longhorn.io/docs/)

## Files in this folder

- `Chart.yaml`: pins the Longhorn chart dependency version (watched by Dependabot).
- `values.yaml`: non-secret values for the chart dependency.
- `longhorn.yaml`: post-install ingress and StorageClass customization.

## Prerequisite

```bash
apt-get install -y open-iscsi nfs-common
```

## Install or upgrade locally (manual)

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm dependency update ./03.longhorn
helm upgrade \
    --install \
    longhorn \
    ./03.longhorn \
    --namespace longhorn-system \
    --create-namespace \
    --values ./03.longhorn/values.yaml \
    --wait
kubectl -n longhorn-system apply -f ./03.longhorn/longhorn.yaml
```

### Confirm installation

```bash
kubectl -n longhorn-system get pods
```

## Automatic updates with Dependabot

Dependabot is configured in `.github/dependabot.yml` to watch `03.longhorn/Chart.yaml`.

When a new `longhorn` chart version is available, Dependabot opens a PR that bumps the pinned dependency in `Chart.yaml`.

## Automatic apply to k3s after merge

Workflow: `.github/workflows/longhorn-upgrade.yml`

Trigger:

- Push to `main` with changes under `03.longhorn/**`
- Manual run (`workflow_dispatch`)

The workflow:
1. Uses a GitHub-hosted runner and joins your tailnet through `tailscale/github-action` with `tag:ci` identity.
2. Configures kubeconfig via `tailscale configure kubeconfig tailscale-operator`.
3. Runs `helm dependency update`, `helm template`, and `helm upgrade --install` for `./03.longhorn` in namespace `longhorn-system`.
4. Applies `03.longhorn/longhorn.yaml` to keep ingress and StorageClass customization in sync.

### Required GitHub repository secrets

- `TS_OAUTH_CLIENT_ID`: Tailscale OAuth client ID for CI tailnet access.
- `TS_OAUTH_SECRET`: Tailscale OAuth client secret for CI tailnet access.
