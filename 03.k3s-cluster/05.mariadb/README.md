# MariaDB (mariadb-operator)

Shared MariaDB server managed by the open-source
[mariadb-operator](https://github.com/mariadb-operator/mariadb-operator). All
WordPress site databases live in this one instance (one database per site) and
are added declaratively as a `Database`/`User`/`Grant` triple under
`cluster/apps/<site>/` - same model as CNPG on `03.postgres/`.

Originally migrated from the docker-host compose stack (Gem Garden) via a
logical dump; the server boots from that dump through `spec.bootstrapFrom.s3`.

## Layout

| Path | What it deploys |
|---|---|
| `operator/` | `HelmRepository` + `mariadb-operator-crds` + `mariadb-operator` charts (ns `mariadb-operator`) |
| `cluster/mariadb.yaml` | The `MariaDB` CR (ns `mariadb`): `mariadb:11.4`, `local-path` 10Gi, bootstrap from S3 dump |
| `cluster/physical-backup.yaml` | Nightly `PhysicalBackup` (mariadb-backup) → `k3s-wordpress-backups/gemgarden/backups`, gzip, retention 14d (03:00 UTC, mirrors CNPG) |
| `cluster/apps/<site>/` | Per-site `Database` + `User` + `Grant` CRs |

Flux order: `mariadb-operator` Kustomization (charts/CRDs) → `mariadb`
Kustomization (CRs), then `gemgarden-wordpress` Kustomization `dependsOn:
mariadb` so the site only deploys once the DB is ready.

## Out-of-band Secrets (one-time, Flux retries until they exist)

```bash
kubectl -n mariadb create secret generic mariadb-root \
  --from-literal=root-password='...'
kubectl -n mariadb create secret generic wordpress-s3-backup-creds \
  --from-literal=ACCESS_KEY_ID='...' \
  --from-literal=SECRET_ACCESS_KEY='...'
kubectl -n mariadb create secret generic gemgarden-db-credentials \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=gemgarden --from-literal=password='...'
```

The `gemgarden-db-credentials` password is authoritative (User CR syncs from
it). The site namespace holds a copy with the **same** password for the
WordPress pod (Keycloak pattern). The S3 credential secret gives object access
to the dedicated `k3s-wordpress-backups` bucket (OCI customer secret keys).

## Adding another site

Create `cluster/apps/<site>/` with the DB/User/Grant triple (copy
`apps/gemgarden/`) + a `Database` whose name matches the site's database, then
deploy the site's own folder (e.g. `07.<site>-wordpress/`). No operator
changes and no extra backups beyond the single nightly `PhysicalBackup`.

## Day-2 operations

```bash
kubectl -n mariadb get mariadb mariadb
kubectl -n mariadb get physicalbackups
kubectl -n mariadb get pods -l app.kubernetes.io/name=mariadb
```

- Connection endpoint for apps: `mariadb.mariadb.svc.cluster.local:3306`.
- Restore after data loss: point a new `MariaDB` CR `bootstrapFrom.s3` at
  `gemgarden/backups` (`backupContentType: Physical`) or `gemgarden/migration`
  (logical) with the desired `targetRecoveryTime`.