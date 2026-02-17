# Production Readiness Audit

> Generated: 2026-02-17
> Repo: `jramsgz/rjLab` — Single-node Talos Kubernetes homelab with ArgoCD GitOps
> Original repo: `qjoly/GitOps` (at `original_repo/GitOps/`)

## Status Legend

- [ ] Not started
- [x] Completed
- [~] Not a bug (same as original repo)

---

## CRITICAL (must fix before deployment)

### 1. Regenerate Cilium CA keys and TLS certificates

- [ ] Fix

`cluster/cilium/install-cilium.yaml` contains base64-encoded `ca.key` and Hubble `tls.key` **copied from the original repo**. Anyone with access to the original repo can decode these and impersonate Cilium components.

**Files:** `cluster/cilium/install-cilium.yaml`

**Fix:**
- Regenerate the Cilium install manifest using `helm template` with fresh CA certs
- Replace the entire `cluster/cilium/install-cilium.yaml` with the new rendered manifest

---

### 2. ~~Align ArgoCD versions~~ — NOT A BUG

- [~] Same as original

Original repo uses the **exact same version split**: v2.13.2 bootstrap → v3.0.5 vault-argocd kustomize overlay → v2.14.21 CMP sidecar images. This is by design:
- `argocd.install.yaml` (v2.13.2) is only the **initial bootstrap** applied via Talos `extraManifests`
- `vault-argocd/kustomization.yaml` (v3.0.5) **replaces it** once ArgoCD is running
- Sidecar images (v2.14.21) only need the `argocd-cmp-server` binary — version doesn't need to match server

---

### 3. Fix ClusterIssuer name mismatch (TLS will fail on all media apps)

- [x] **FIXED**

Original repo uses `cloudflare` everywhere. Our repo had `letsencrypt-prod` in all 5 media apps.

**Fixed files:**
- `cluster/apps/media/jackett.yaml` — `letsencrypt-prod` → `cloudflare`
- `cluster/apps/media/kyoo.yaml` — same
- `cluster/apps/media/qbittorrent.yaml` — same
- `cluster/apps/media/radarr.yaml` — same
- `cluster/apps/media/sonarr.yaml` — same

---

### 4. ~~AVP plugin missing on Jackett and Kyoo~~ — NOT A BUG

- [~] Same as original

The parent ApplicationSet (`as-app.yml`) processes each `cluster/apps/*` directory with `argocd-vault-plugin-kustomize`. This resolves ALL `<path:kv/...>` placeholders in the Application YAML files **before** they are applied as ArgoCD resources. Individual apps don't need their own AVP plugin declaration. Original repo uses the exact same pattern.

---

## HIGH (should fix before deployment)

### 5. ~~Fix `VAULT_ADDR` missing protocol prefix~~ — SKIPPED

- [~] Not an issue per user

The `vault-credentials.yaml.tmpl` is a template — the actual secret is created via the `kubectl create secret` command in the docs, which correctly includes `http://`.

---

### 6. ~~Fix OpenEBS StorageClass missing fields~~ — NOT A BUG

- [~] Same as original

Original mocha `bootstrap/openebs.yaml` uses the **exact same StorageClass** without `volumeBindingMode`, `reclaimPolicy`, or `allowVolumeExpansion`. Works fine for single-node.

---

### 7. ~~Resolve duplicate ArgoCD ConfigMap~~ — NOT A BUG

- [~] Same as original

Original uses the same split: `argocd.config.yaml` (full settings) + `vault-argocd/config.yaml` (resource.exclusions only). `patchesStrategicMerge` **merges** — it doesn't replace. The base settings (admin.enabled, exec.enabled, timeout.reconciliation) are preserved.

---

### 8. ~~Fix Vault HA for single-node cluster~~ — NOT A BUG

- [~] Same as original

Original `common/vault.yaml` uses the **exact same config** (HA Raft, no explicit replicas). Default is 1 replica. HA mode with Raft and 1 replica works fine.

---

### 9. Standardize LoadBalancer sharing keys

- [x] **FIXED**

Original Traefik uses `sharing-key: "public"`. qBittorrent (our custom addition) was using `"default"`. Fixed to `"public"` so both share the same LB IP.

---

## MEDIUM (improve before or shortly after deployment)

### 10. ~~Pin image tags~~ — NOT A BUG

- [~] Same as original

Original also uses `tag: "latest"` for both Byparr (in Jackett) and Kromgo. Same pattern.

---

### 11. Parameterize hardcoded email

- [x] **FIXED**

Changed `contact.makerlab@gmail.com` → `<path:kv/cluster#email>` in `cluster/system/misc/cluster-issuer.yaml`.
Also added `email` to the `kv/cluster` entry in README's Vault table.

---

### 12. Fix documentation inconsistencies

- [x] **FIXED**

| File | Fix |
|------|-----|
| `docs/backup.md` | Fixed secret name (`restic-credentials` → `restic-base-credentials`) and namespace (`kube-system` → `volsync-system`) |
| `docs/secret-management.md` | Fixed API version (`v1beta1` → `v1`) and ClusterIssuer (`letsencrypt-prod` → `cloudflare`) |

---

### 13. Update README Vault secrets table

- [x] **FIXED**

Added missing Vault paths: `kv/kyoo`, `kv/sonarr`, `kv/radarr` and added `email` to `kv/cluster`.

---

## LOW (nice to have improvements)

### 14. Consider Vault standalone mode

- [ ] Evaluate

Since you have a single node, Vault standalone mode is simpler. But HA Raft with 1 replica (current config) also works fine and is the same pattern as the original.

---

### 15. Add active VolSync backup ReplicationSources

- [ ] Implement

`cluster/system/backup/example-volsync.yaml` is only a template. No PVCs are being backed up yet. Create `ReplicationSource` resources for critical app PVCs (sonarr, radarr, kyoo).

---

### 16. Reduce vendored manifest maintenance burden

- [ ] Implement

Consider adding Taskfile tasks to regenerate `install-cilium.yaml` and `argocd.install.yaml` so upgrades are reproducible.

---

## Post-Fix Verification Checklist

- [ ] `task gen-config && task apply-config` — Talos config generates and applies without errors
- [ ] `task bootstrap` — cluster bootstraps with ArgoCD coming up cleanly
- [ ] `kubectl get clusterissuer` — `cloudflare` issuer is Ready
- [ ] `kubectl get certificate -A` — certificates issued for all ingresses
- [ ] `kubectl get application -n argocd` — all apps Synced/Healthy
- [ ] `kubectl get secret -n cilium cilium-ca` — new CA cert differs from original repo
- [ ] `vault status` — Vault initialized and unsealed
- [ ] Populate `kv/cluster` with `email` key in Vault
- [ ] Test each app's ingress URL — TLS works with valid Let's Encrypt certs
