## Backup System

Backups use [Volsync](https://github.com/backube/volsync) to replicate PVCs via [Restic](https://restic.net/) to a local [Minio](https://min.io/) instance running on the second HDD.

### Architecture

```
PVC → Volsync ReplicationSource → Restic → Minio (hostPath /var/minio-data on 2nd HDD)
```

- **Minio** runs as a Deployment with a `hostPath` volume at `/var/minio-data` (see `cluster/system/minio/`)
- **Volsync** is installed via Helm chart (see `cluster/system/storage/volsync.yaml`)
- **Restic credentials** are stored in Vault and synced via ExternalSecret (see `cluster/system/backup/restic-credentials.yaml`)

### Restic Credentials in Vault

Store credentials at `kv/restic` in Vault:

```bash
vault kv put kv/restic \
  AWS_ACCESS_KEY_ID="minioadmin" \
  AWS_SECRET_ACCESS_KEY="your-minio-password" \
  RESTIC_PASSWORD="your-restic-encryption-password" \
  RESTIC_REPOSITORY="s3:http://minio.minio.svc.cluster.local:9000/backups"
```

The ExternalSecret in `cluster/system/backup/restic-credentials.yaml` syncs these to a Kubernetes Secret named `restic-base-credentials` in the `volsync-system` namespace.

### Initialize a Restic Repository

Before creating your first backup, initialize the restic repository in Minio:

```bash
kubectl port-forward -n minio svc/minio 9000:9000 &

export AWS_ACCESS_KEY_ID=minioadmin
export AWS_SECRET_ACCESS_KEY=your-minio-password
export RESTIC_PASSWORD=your-restic-encryption-password
export RESTIC_REPOSITORY=s3:http://localhost:9000/backups

restic init
```

### Create a Backup (ReplicationSource)

To back up a PVC, create a `ReplicationSource` in the same namespace:

```yaml
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: my-app-backup
  namespace: my-app
spec:
  sourcePVC: my-app-data
  trigger:
    schedule: "0 */6 * * *"
  restic:
    pruneIntervalDays: 7
    repository: restic-credentials
    retain:
      hourly: 6
      daily: 5
      weekly: 4
      monthly: 2
      yearly: 1
    copyMethod: Direct
```

A commented example is at `cluster/system/backup/example-volsync.yaml`.

### Verify Backups

```bash
kubectl port-forward -n minio svc/minio 9000:9000 &
restic snapshots
```

### Restore from Backup

Create a `ReplicationDestination`:

```yaml
apiVersion: volsync.backube/v1alpha1
kind: ReplicationDestination
metadata:
  name: my-app-restore
  namespace: my-app
spec:
  trigger:
    manual: restore-once
  restic:
    repository: restic-credentials
    destinationPVC: my-app-data
    copyMethod: Direct
```