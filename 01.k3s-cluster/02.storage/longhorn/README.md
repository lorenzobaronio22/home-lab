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

## Automatic updates with Dependabot

Dependabot is configured in `.github/dependabot.yml` to watch `01.k3s-cluster/02.storage/longhorn/Chart.yaml`.

When a new `longhorn` chart version is available, Dependabot opens a PR that bumps the pinned dependency in `Chart.yaml`.

## Automatic apply to k3s after merge

Workflow: `.github/workflows/longhorn-upgrade.yml`

Trigger:

- Push to `main` with changes under `01.k3s-cluster/02.storage/longhorn/**`
- Manual run (`workflow_dispatch`)

The workflow:
1. Uses a GitHub-hosted runner and joins your tailnet through `tailscale/github-action` with `tag:ci` identity.
2. Configures kubeconfig via `tailscale configure kubeconfig tailscale-operator`.
3. Runs `helm dependency update`, `helm template`, and `helm upgrade --install` for `./01.k3s-cluster/02.storage/longhorn` in namespace `longhorn-system`.
4. Applies `01.k3s-cluster/02.storage/longhorn/longhorn.yaml` to keep ingress and StorageClass customization in sync.

### Required GitHub repository secrets

- `TS_OAUTH_CLIENT_ID`: Tailscale OAuth client ID for CI tailnet access.
- `TS_OAUTH_SECRET`: Tailscale OAuth client secret for CI tailnet access.
