# Keycloak (Identity)

Single-instance Keycloak 27 managed by the official
[Keycloak Operator](https://www.keycloak.org/operator/installation). Data lives
in the shared CloudNativePG cluster (see `03.postgres/`); TLS is terminated at
the Tailscale edge, so Keycloak runs plain HTTP internally. Exposed only on the
tailnet at `keycloak.tail10187.ts.net` (MagicDNS).

## Layout

| Path | What it deploys |
|---|---|
| `keycloak-operator/` | `GitRepository` pointing at `keycloak/keycloak-k8s-resources` (pinned tag) + the Flux `Kustomization` that applies the upstream `kubernetes/` manifests (installs the operator and its CRDs into `keycloak`) |
| `keycloak/` | The `Keycloak` CR (v2beta1) + the repo-owned Tailscale `Ingress` |

Flux order: `postgres → identity → apps`. Inside identity, `keycloak-operator`
must be ready before `keycloak` applies its CR (CRDs are created by the
operator install).

## Why these choices

- **Operator, not a Helm chart**: the Keycloak project ships no Helm chart; the
  operator + pinned kustomize manifests is the supported path and matches the
  CNPG/Tailscale-operator GitOps pattern here.
- **Tailscale edge TLS** (`http.httpEnabled`, `proxy.headers: xforwarded`,
  fixed `hostname`): the operator's own Ingress is disabled because it can't
  express Tailscale's short-host convention; the Ingress in this folder wires
  `keycloak` → `keycloak.tail10187.ts.net`.
- **No PV**: Keycloak is stateless; Barman already backs up its schema.

## Secrets (one-time, out-of-band)

```bash
kubectl -n postgres create secret generic keycloak-db-credentials \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=keycloak --from-literal=password='...'
kubectl -n keycloak create secret generic keycloak-db-credentials \
  --from-literal=username=keycloak --from-literal=password='...'
kubectl -n keycloak create secret generic keycloak-admin \
  --from-literal=username=admin --from-literal=password='...'
```

The `postgres` Secret is consumed by the CNPG `DatabaseRole` CR and must be
`kubernetes.io/basic-auth` with both `username` and `password` keys.

The admin Secret name **must not** be `keycloak-initial-admin` — that is the
name the operator itself generates when no custom admin secret is configured;
pointing `bootstrapAdmin.user.secret` at it makes the operator try to create a
colliding Secret and fail with `409 AlreadyExists` (StatefulSet never created).
Use any *other* name (the CR defaults to `keycloak-admin`). If you previously
pre-created `keycloak-initial-admin`, rename it:

```bash
kubectl -n keycloak get secret keycloak-initial-admin -o yaml \
  | sed 's/name: keycloak-initial-admin/name: keycloak-admin/' | kubectl apply -f -
kubectl -n keycloak delete secret keycloak-initial-admin
```

Until they exist, the `postgres` deploy of the DB role and the Keycloak CR
report unready and Flux retries (same pattern as `cloudflared`).

## Updates

- **Operator + server**: Renovate bumps the `keycloak-k8s-resources`
  `ref.tag` in `gitrepository.yaml` (grouped as "keycloak operator"). The
  operator and the Keycloak server image are released in lockstep, so one bump
  upgrades both and the operator rolls the StatefulSet. **Do not** pin
  `spec.image` on the CR — it desyncs server/operator versions.
- Keycloak runs one-way DB schema migrations on startup; review major-version
  PRs before merging (no auto-downgrade without a DB restore).

## Day-2

- Admin console: `https://keycloak.tail10187.ts.net/admin` (credentials from the
  `keycloak-admin` Secret).
- Change the initial password and enable MFA for the admin user before treating
  it as production.