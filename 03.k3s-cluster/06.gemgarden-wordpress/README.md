# Gem Garden WordPress

The `gemgarden.org` (and `www.gemgarden.org`) WordPress site, migrated from the
docker-host compose stack. TLS is terminated at the Cloudflare edge - there is
**no Ingress**: the ClusterIP Service is the origin of the cluster Cloudflare
Tunnel's Public Hostname routes (dashboard-managed, see
`01.networking/cloudflared/README.md`).

## Layout

| Path | What it deploys |
|---|---|
| `namespace.yaml` | `gemgarden-wordpress` |
| `wp-config.yaml` | `gemgarden-wp-config` ConfigMap: wp-config reads DB creds/salts from pod env and maps Cloudflare `X-Forwarded-Proto` → HTTPS |
| `pvc.yaml` | `gemgarden-wp-data` (local-path, 2Gi) for `/var/www/html` |
| `seed-job.yaml` | One-time Job: copies node-staged docker `wp_data` (hostPath `/mnt/staging/gemgarden-wp`) into the PVC, chowns to www-data (33) |
| `deployment.yaml` | `gemgarden-wordpress` (wordpress:latest, probes, resources, Recreate strategy) |
| `service.yaml` | ClusterIP `:80` → tunnel origin |

The DB lives in the separate `mariadb` namespace (`mariadb.mariadb.svc.cluster.local`); the
site's `Database`/`User`/`Grant` are declared in `05.mariadb/cluster/apps/gemgarden/`.

## Out-of-band Secrets (one-time)

```bash
kubectl -n gemgarden-wordpress create secret generic gemgarden-wp-credentials \
  --from-literal=username=gemgarden \
  --from-literal=password='...' \
  --from-literal=auth-key='...' \
  --from-literal=secure-auth-key='...' \
  --from-literal=logged-in-key='...' \
  --from-literal=nonce-key='...' \
  --from-literal=auth-salt='...' \
  --from-literal=secure-auth-salt='...' \
  --from-literal=logged-in-salt='...' \
  --from-literal=nonce-salt='...'
```

`password` must match `gemgarden-db-credentials` in the `mariadb` namespace.
Generate the 8 salt values with `openssl rand -hex 32` (or wp-cli's `wp
config shuffle-salts` output). wp-config.php is generated from these env vars,
kept out of git.

## Migration notes

- The DB is created by bootstrapping the `MariaDB` CR from the logical dump
  (`mariadb-wordpress` bucket, `gemgarden/migration/` prefix) - no data loss.
- `wp-config.php` is **excluded** from the staged file copy; the ConfigMap
  version is authoritative.
- HTTPS behind Cloudflare requires the `X-Forwarded-Proto` mapping in
  wp-config; without it WordPress serves mixed-content/wrong canonical URLs.
- Upgrade WordPress by bumping the Deployment image or on a maintenance
  window; the content lives in the PVC, the DB in the shared MariaDB server
  (both backed up).