# Getting Started

## Prerequisites

- A bare-metal machine (amd64) with 2 HDDs:
  - **HDD 1**: Storage disk (OpenEBS LVM)
  - **HDD 2**: Backup disk (Minio)
- A domain managed by Cloudflare
- [talosctl](https://www.talos.dev/latest/talos-guides/install/talosctl/) installed on your workstation
- [task](https://taskfile.dev/installation/) installed on your workstation
- [kubectl](https://kubernetes.io/docs/tasks/tools/) installed on your workstation

## Step 1: Get a Talos Image

Go to the [Talos Image Factory](https://factory.talos.dev/) and create a schematic with these extensions:

- `siderolabs/iscsi-tools` (required for OpenEBS LVM)
- `siderolabs/util-linux-tools` (useful utilities)

Download the ISO or raw disk image for your architecture (amd64). Save your **Schematic ID** — you'll need it for upgrades.

## Step 2: Install Talos on the Machine

### Option A: USB Boot
Write the ISO to a USB drive and boot from it:
```bash
dd if=talos-amd64.iso of=/dev/sdX bs=4M status=progress
```

### Option B: Rescue Mode (e.g., OVH)
If your server is hosted, boot into rescue mode and write the image directly:
```bash
wget https://factory.talos.dev/image/<SCHEMATIC_ID>/v1.9.5/metal-amd64.raw.xz
xz -d metal-amd64.raw.xz
dd if=metal-amd64.raw of=/dev/sda bs=4M status=progress
```
Reboot from disk after writing.

## Step 3: Configure and Deploy

1. **Fork/clone this repo**

2. **Create your environment file**:
   ```bash
   cp .env.example .env
   # Edit .env with your node IP, cluster name, schematic ID
   ```

3. **Generate and apply the Talos config**:
   ```bash
   task gen-config      # Generates controlplane.yaml with all patches
   task apply-config    # Applies config to the node (first time: --insecure)
   ```

4. **Bootstrap the cluster** (first time only):
   ```bash
   task bootstrap       # Initializes etcd and the control plane
   task kubeconfig      # Downloads kubeconfig to ~/.kube/config
   ```

5. **Wait for self-provisioning**:
   - Talos boots → extraManifests install Cilium + ArgoCD + metrics-server
   - ArgoCD reads `bootstrap.yml` → deploys `cluster/bootstrap/` (ArgoCD config, cert-manager, Vault, ApplicationSets, OpenEBS)
   - ApplicationSets discover and deploy everything else

   > **Note**: Talos `extraManifests` installs the upstream ArgoCD manifests directly.
   > Once the bootstrap ArgoCD Application syncs, it replaces them with the kustomized
   > version (which includes AVP sidecars and the `server.insecure` ConfigMap patch).

## Step 4: Set Up Storage

### Storage HDD (OpenEBS LVM)

After the cluster is running, create an LVM volume group on the storage disk:

```bash
# Find your storage disk
talosctl get disks --nodes <NODE_IP> --endpoints <NODE_IP> --talosconfig cluster/talos/talosconfig

# Create a namespace with privileged PodSecurity (required for host access)
kubectl create namespace debug
kubectl label namespace debug pod-security.kubernetes.io/enforce=privileged

# Create a debug pod with LVM tools and host /dev access
# (chroot won't work on Talos — it has no shell on the host rootfs)
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: lvm-setup
  namespace: debug
spec:
  hostPID: true
  containers:
  - name: lvm-setup
    image: ubuntu:24.04
    command: ["bash", "-c", "apt-get update && apt-get install -y lvm2 && echo READY && sleep 3600"]
    securityContext:
      privileged: true
    volumeMounts:
    - name: dev
      mountPath: /dev
    - name: run-udev
      mountPath: /run/udev
  volumes:
  - name: dev
    hostPath:
      path: /dev
  - name: run-udev
    hostPath:
      path: /run/udev
  restartPolicy: Never
EOF

# Wait for lvm2 to be installed
kubectl wait --for=condition=Ready pod/lvm-setup -n debug --timeout=180s

# Create the LVM physical volume and volume group
kubectl exec lvm-setup -n debug -- pvcreate /dev/sdb
kubectl exec lvm-setup -n debug -- vgcreate lvmvg /dev/sdb

# Clean up
kubectl delete pod lvm-setup -n debug
kubectl delete namespace debug
```

The OpenEBS ArgoCD Application (`cluster/bootstrap/openebs.yaml`) will create the `openebs-lvmpv` StorageClass automatically.

### Backup HDD (Minio)

The backup disk is mounted via the Talos patch `cluster/patches/backup-disk.yml`. Ensure the disk is formatted and the mount path matches:

```bash
# Format the backup disk (if needed) via debug pod
# mkfs.ext4 /dev/sdc
# mkdir -p /var/minio-data
# mount /dev/sdc /var/minio-data
```

The Minio deployment (`cluster/system/minio/`) uses a `hostPath` volume pointing to `/var/minio-data`.

## Step 5: Initialize Vault

Vault must be initialized and unsealed manually on first boot:

```bash
# Wait for Vault pod to be running
kubectl get pods -n vault -w

# Initialize Vault
task vault-init
# This outputs cluster-keys.json — SAVE THIS SECURELY

# Unseal Vault (joins and unseals all replicas)
task vault-unseal

# Enable KV v1 secrets engine
ROOT_TOKEN=$(cat cluster-keys.json | jq -r ".root_token")
kubectl exec -n vault vault-0 -- sh -c "VAULT_TOKEN=$ROOT_TOKEN vault secrets enable -path=kv -version=1 kv"

# Create the Vault token secret for External Secrets
# (namespace may not exist yet — ApplicationSets haven't synced)
kubectl create namespace external-secrets --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic vault-token \
  --namespace external-secrets \
  --from-literal=token=$ROOT_TOKEN
```

## Step 6: Create ArgoCD Vault Credentials

Create the vault-credentials secret so the AVP sidecars (already deployed via bootstrap) can connect to Vault:

```bash
# Create the vault-credentials secret for AVP
ROOT_TOKEN=$(cat cluster-keys.json | jq -r ".root_token")
kubectl create secret generic vault-credentials \
  --namespace argocd \
  --from-literal=AVP_AUTH_TYPE=token \
  --from-literal=AVP_KV_VERSION=1 \
  --from-literal=AVP_TYPE=vault \
  --from-literal=VAULT_ADDR=http://vault.vault.svc.cluster.local:8200 \
  --from-literal=VAULT_TOKEN=$ROOT_TOKEN
```

The ArgoCD `repo-server` is deployed with AVP CMP sidecars from bootstrap. Once this secret is created, the sidecars will start resolving `<path:kv/...>` placeholders in manifests at sync time.

## Step 7: Populate Vault Secrets

```bash
# Port-forward Vault
task vault-port-forward &

export VAULT_ADDR=http://localhost:8200
export VAULT_TOKEN=$(cat cluster-keys.json | jq -r ".root_token")

# Cluster basics
vault kv put kv/cluster domain="yourdomain.com" IP="192.168.1.100" email="you@example.com"

# Cloudflare DNS token
vault kv put kv/cloudflare dnsToken="your-cloudflare-api-token"

# Grafana
vault kv put kv/grafana user="admin" pass="your-grafana-password" \
  prometheus_user="prometheus" prometheus_pass="your-prom-password"

# Minio
vault kv put kv/minio rootUser="minioadmin" rootPassword="your-minio-password"

# Restic backup credentials (use Minio creds)
vault kv put kv/restic \
  AWS_ACCESS_KEY_ID="minioadmin" \
  AWS_SECRET_ACCESS_KEY="your-minio-password" \
  RESTIC_PASSWORD="your-restic-encryption-password" \
  RESTIC_REPOSITORY="s3:http://minio.minio.svc.cluster.local:9000/backups"

# --- App secrets (add when deploying apps) ---

# Kyoo (media server)
vault kv put kv/kyoo apikey="your-kyoo-api-key" tmdb="your-tmdb-api-key"

# Sonarr
vault kv put kv/sonarr \
  token="your-sonarr-api-key" \
  url="http://sonarr.sonarr.svc.cluster.local:8989"

# Radarr
vault kv put kv/radarr \
  token="your-radarr-api-key" \
  url="http://radarr.radarr.svc.cluster.local:7878"
```

## Step 8: Verify

```bash
# Check cluster health
task health

# Check ArgoCD applications
kubectl get applications -n argocd

# Access ArgoCD UI (get initial admin password)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
# Then access via the ingress: https://argocd.<your-domain>
```

## Upgrading

### Talos
```bash
task upgrade-talos -- v1.10.0
```

### Kubernetes
```bash
task upgrade-k8s -- v1.33.0
```
