# PostgreSQL (CloudNativePG)

Single-instance PostgreSQL 17 server managed by the
[CloudNativePG](https://cloudnative-pg.io) operator. All application databases
(Keycloak, custom web apps, ...) live in this one cluster and are added
declaratively — see [Adding a database](#adding-a-database).

Backups go to OCI Object Storage via its S3 Compatibility API using the
[Barman Cloud plugin](https://cloudnative-pg.github.io/plugin-barman-cloud/):
continuous WAL archiving + a nightly base backup at 03:00 UTC, retention 14d.
Data is **not** expected to survive VM rebuilds — it is rebuilt from backups.

## Layout

| Path | What it deploys |
|---|---|
| `../02.cert-manager/` | cert-manager (prerequisite of the barman plugin TLS) |
| `cnpg-operator/` | CloudNativePG operator (`cnpg-system`) |
| `barman-plugin/` | Barman Cloud plugin HelmRelease (`cnpg-system`) |
| `workload/objectstore.yaml` | `ObjectStore` CR pointing at OCI S3 |
| `workload/cluster/` | The `Cluster` CR + nightly `ScheduledBackup` |
| `workload/pgweb/` | pgweb web client, exposed on the tailnet at `pgweb.tail10187.ts.net` |

Flux order: `networking → cert-manager → databases → postgres → apps`.
The `databases` Kustomization installs the operator + plugin (creating the
CRDs); the `postgres` Kustomization applies the CRs that use them.

## One-time OCI setup

1. Create a bucket in your home region, e.g. `k3s-postgres-backups`.
2. Create an IAM policy allowing your user to manage objects in that bucket.
3. Console → Profile → **Customer secret keys** → generate key; copy the  
   Access Key and Secret Key (the secret is shown only once).
4. Find your tenancy's object storage namespace and region identifier.
5. Edit `barman-plugin/objectstore.yaml`: replace `<OCI_NAMESPACE>` and
   `<OCI_REGION>` in the endpoint URL.
6. Create the credentials Secret manually (kept out of git, same pattern as
   the Tailscale OAuth secret):

   ```bash
   kubectl -n postgres create secret generic oci-backup-creds \
     --from-literal=ACCESS_KEY_ID='<access key>' \
     --from-literal=SECRET_ACCESS_KEY='<secret key>'
   ```

Notes:
- The endpoint must **not** include the bucket name — including it breaks
  backup listing while uploads still succeed.
- `AWS_REGION=us-east-1` is set on the plugin pod (see
  `barman-plugin/helmrelease.yaml`) because boto needs *a* region; the
  endpoint URL selects the actual OCI region.

## Day-2 operations

```bash
# Cluster status
kubectl -n postgres get cluster postgres

# List backups in OCI
kubectl cnpg backup postgres -n postgres          # on-demand base backup
kubectl -n postgres get backups

# Connection endpoints (in-cluster only)
postgres-rw.postgres.svc.cluster.local   # read/write
postgres-ro.postgres.svc.cluster.local   # read-only
```

The pgweb client (superuser credentials preloaded from the
`postgres-superuser` Secret) is available at
`https://pgweb.tail10187.ts.net` for anyone on the tailnet.

### Restoring after a VM rebuild

1. Rebuild k3s + Flux per `../README.md`; Flux recreates the operator,
   plugin and ObjectStore. Recreate the `oci-backup-creds` Secret (step 6
   above) **before** applying the Cluster, or let Flux retry until it exists.
2. Apply a recovery Cluster (do **not** reuse the old cluster name):

   ```yaml
   apiVersion: postgresql.cnpg.io/v1
   kind: Cluster
   metadata:
     name: postgres-restore
     namespace: postgres
   spec:
     instances: 1
     storage:
       size: 10Gi
       storageClass: local-path
     bootstrap:
       recovery:
         source: oci-backup
     plugins:
       - name: barman-cloud.cloudnative-pg.io
         parameters:
           barmanObjectName: oci-backup
           recoveryWindow: "14d" # optional PITR window
   ```

3. Verify data, then rename the restored cluster back to `postgres`
   (scale down apps first if they point at `-rw` service names).

### Adding a database

Each app gets its own database + user, declared in git. Example for Keycloak:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Database
metadata:
  name: keycloak
  namespace: postgres
spec:
  clusterRef:
    name: postgres
  name: keycloak
  owner: keycloak        # role is created declaratively too
---
apiVersion: postgresql.cnpg.io/v1
kind: Role
metadata:
  name: keycloak
  namespace: postgres
spec:
  clusterRef:
    name: postgres
  name: keycloak
  login: true
  passwordSecret:
    name: keycloak-db-credentials
```

Then create the password Secret and point the app at:

```
postgres-rw.postgres.svc.cluster.local:5432/keycloak
```

with credentials from the app's Secret.
