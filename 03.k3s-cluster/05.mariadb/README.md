# MariaDB (mariadb-operator)

Shared MariaDB server managed by the open-source
[mariadb-operator](https://github.com/mariadb-operator/mariadb-operator). All
WordPress site databases live in this one instance (one database per site) and
are added declaratively as a `Database`/`User`/`Grant` triple under
`cluster/apps/<site>/` - same model as CNPG on `03.postgres/`.

Originally migrated from the docker-host compose stack (Gem Garden) via a
logical dump; since that first boot the data lives on the PVC and the
`bootstrapFrom` block has been removed (the migration dump is gone too).
Recovery is now PhysicalBackup-only — see [Restoring](#restoring-after-data-loss).

## Layout

| Path | What it deploys |
|---|---|
| `operator/` | `HelmRepository` + `mariadb-operator-crds` + `mariadb-operator` charts (ns `mariadb-operator`) |
| `cluster/mariadb.yaml` | The `MariaDB` CR (ns `mariadb`): `mariadb:11.4`, `local-path` 10Gi |
| `cluster/physical-backup.yaml` | Nightly `PhysicalBackup` (`physicalbackup-daily`, mariadb-backup) → `homelab-backups/mariadb/gemgarden/backups`, gzip, retention 14d (03:00 UTC, mirrors CNPG) |
| `cluster/apps/<site>/` | Per-site `Database` + `User` + `Grant` CRs |

Flux order: `mariadb-operator` Kustomization (charts/CRDs) → `mariadb`
Kustomization (CRs), then `gemgarden-wordpress` Kustomization `dependsOn:
mariadb` so the site only deploys once the DB is ready.

## Backups

Backups share the `homelab-backups` bucket with the postgres stack
(`../03.postgres/`); this workload owns the `mariadb/` prefix. A
`PhysicalBackup` covers the **whole instance** (all site databases), so the
`gemgarden/` level is a naming choice, not a scope limit — a second site adds
prefixes, not backup jobs.

Force an out-of-band backup by bumping `spec.schedule.onDemand` in
`cluster/physical-backup.yaml`; the operator fires a job whenever the
identifier differs from `status.lastScheduleOnDemand`.

## Out-of-band Secrets (one-time, Flux retries until they exist)

```bash
kubectl -n mariadb create secret generic mariadb-root \
  --from-literal=root-password='...'
kubectl -n mariadb create secret generic homelab-backup-creds \
  --from-literal=ACCESS_KEY_ID='...' \
  --from-literal=SECRET_ACCESS_KEY='...'
kubectl -n mariadb create secret generic gemgarden-db-credentials \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=gemgarden --from-literal=password='...'
```

The `gemgarden-db-credentials` password is authoritative (User CR syncs from
it). The site namespace holds a copy with the **same** password for the
WordPress pod (Keycloak pattern). The S3 credential secret is the same OCI
customer secret key used by the `postgres` namespace — copy it rather than
generating a new one, and keep the access key ID and secret key paired (a
mismatched pair fails with `SignatureDoesNotMatch`).

Secrets must exist **before** the manifests that reference them: a missing
credential Secret leaves the `PhysicalBackup` job failing silently at 03:00
UTC, and a missing one in `postgres` crash-loops the CNPG instance pod.

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

### Restoring after data loss

Physical backups are the only restore path (the logical migration dump is
gone). Point a **new** `MariaDB` CR at the `PhysicalBackup` — never reuse the
old cluster name:

```yaml
spec:
  bootstrapFrom:
    backupRef:
      name: physicalbackup-daily
      kind: PhysicalBackup
```

If the backup fails, read the job log before the operator garbage-collects it:

```bash
kubectl -n mariadb logs job/<job-name> --tail=200
```