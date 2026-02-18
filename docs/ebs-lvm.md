# Storage (OpenEBS LVM)

This cluster uses [OpenEBS LVM LocalPV](https://openebs.io/docs/user-guides/local-storage-user-guide/local-pv-lvm/lvm-localpv) to provide persistent storage backed by an LVM volume group on the storage HDD.

## Setup

### 1. Create the LVM Volume Group

After installing Talos and bootstrapping the cluster, create an LVM volume group on the storage disk. Since Talos is immutable, use a privileged debug pod:

```bash
# First, identify the storage disk
talosctl -n <NODE_IP> disks

# Launch a privileged pod
kubectl run --rm -it lvm-setup --image=ubuntu:24.04 --privileged --overrides='
{
  "spec": {
    "hostPID": true,
    "containers": [{
      "name": "lvm-setup",
      "image": "ubuntu:24.04",
      "command": ["chroot", "/host", "bash"],
      "stdin": true,
      "tty": true,
      "securityContext": {"privileged": true},
      "volumeMounts": [{"name": "host", "mountPath": "/host"}]
    }],
    "volumes": [{"name": "host", "hostPath": {"path": "/"}}]
  }
}'

# Inside the pod:
apt-get update && apt-get install -y lvm2
pvcreate /dev/sdb       # Replace with your storage disk
vgcreate lvmvg /dev/sdb
vgdisplay lvmvg         # Verify
exit
```

### 2. Deploy OpenEBS

OpenEBS is deployed via ArgoCD Application in `cluster/bootstrap/openebs.yaml`. It installs:

- **OpenEBS LVM LocalPV** controller and node agent
- **StorageClass** `openebs-lvmpv` (default) backed by the `lvmvg` volume group

```yaml
# The StorageClass created:
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: openebs-lvmpv
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: local.csi.openebs.io
parameters:
  storage: "lvm"
  volgroup: "lvmvg"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
allowVolumeExpansion: true
```

### 3. Use in Applications

Simply create a PVC — it will use the default `openebs-lvmpv` StorageClass:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## Monitoring

Check LVM volume group space:

```bash
# Via debug pod
kubectl run --rm -it lvm-check --image=ubuntu:24.04 --privileged --overrides='...'
# Inside: vgs lvmvg
```

Check OpenEBS volumes:

```bash
kubectl get lvmvolumes -A
```

## Notes

- LVM volumes are **node-local** — PVCs are bound to the node where the volume group resides
- `WaitForFirstConsumer` binding mode ensures volumes are created on the correct node
- Volume expansion is supported (`allowVolumeExpansion: true`)
- For backups, use Volsync — see [Backup](backup.md)