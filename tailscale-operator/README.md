## Kubernetes Operator

[Docs references](https://tailscale.com/docs/features/kubernetes-operator)

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
