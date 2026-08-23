# Tailscale Operator

[Docs references](https://tailscale.com/docs/features/kubernetes-operator)

Deployed by Flux as a `HelmRelease` pulling the upstream chart directly — no umbrella chart or
vendored `.tgz` needed anymore.

## Files in this folder

- `helmrepository.yaml`: points Flux at `https://pkgs.tailscale.com/helmcharts`.
- `helmrelease.yaml`: pins the chart version and wires values + OAuth secret.
- `values.yaml`: non-secret operator values (rendered into a ConfigMap by kustomize). The operator registers its API server proxy node as `tailscale-operator-oci` on the tailnet.
- `proxygroup.yaml`: ProxyGroup manifest for ingress high availability, applied automatically with the release.
- `nginx-hello.yaml`: optional example service exposure (not applied by Flux; manual only).

## OAuth credentials

The HelmRelease reads them from the `tailscale-operator-oauth` Secret in the `tailscale` namespace:

```bash
kubectl -n tailscale create secret generic tailscale-operator-oauth \
  --from-literal=client-id="<operator-oauth-client-id>" \
  --from-literal=client-secret="<operator-oauth-client-secret>"
```

This is a documented one-time step in the [rebuild runbook](../../README.md). If the Secret is
missing the HelmRelease reports an error and retries every minute until it exists.

## Automatic updates

Renovate watches the `version:` pin in `helmrelease.yaml` (config in `.github/renovate.json`) and
opens a PR when a new chart version is released. Merging it makes helm-controller upgrade the
release in-cluster. There is no deploy workflow anymore.

## Verify

```bash
flux get helmrelease tailscale-operator -n tailscale
kubectl -n tailscale get pods
```

The operator node `tailscale-operator-oci` appears in the Tailscale admin console; its API-server
proxy can still be used for read-only cluster access from other tailnet devices.

## [Optional] Expose an NGINX example service

[Documentation Reference](https://tailscale.com/docs/features/kubernetes-operator/how-to/cluster-ingress)

```bash
kubectl apply -f 03.k3s-cluster/01.networking/tailscale-operator/nginx-hello.yaml
kubectl delete -f 03.k3s-cluster/01.networking/tailscale-operator/nginx-hello.yaml
```
