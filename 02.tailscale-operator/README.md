## Kubernetes Operator

[Docs references](https://tailscale.com/docs/features/kubernetes-operator)

## Install Operator

```bash
helm repo add tailscale https://pkgs.tailscale.com/helmcharts
helm repo update
export CLIENT_ID=<OAauth client ID>
export CLIENT_SECRET=<OAuth client secret>
helm upgrade \
  --install \
  tailscale-operator \
  tailscale/tailscale-operator \
  --namespace=tailscale \
  --create-namespace \
  --set-string oauth.clientId="$CLIENT_ID" \
  --set-string oauth.clientSecret="$CLIENT_SECRET" \
  --set-string apiServerProxyConfig.mode="true" \
  --wait
```

## Setup ProxyGroup for ingress HA

```bash
kubectl apply -f tailscale.yaml
```

## [Optional] Expose a NGINX example service

[Documentation Reference](https://tailscale.com/docs/features/kubernetes-operator/how-to/cluster-ingress)

```bash
kubectl apply -f nginx-hello.yaml
```
