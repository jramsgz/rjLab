# Welcome

Documentation for my HomeLab GitOps repository — a single Kubernetes cluster managed declaratively with ArgoCD on bare-metal Talos Linux.

## Architecture

- **Single node** amd64 server running [Talos Linux](https://talos.dev/) v1.9.5
- **2x 1TB HDDs**: one for persistent storage (OpenEBS LVM), one for backups (Minio)
- **GitOps**: ArgoCD auto-syncs this repository to the cluster
- **Secrets**: HashiCorp Vault + External Secrets Operator + ArgoCD Vault Plugin

## Technology Stack

| Component | Tool |
|-----------|------|
| OS | [Talos Linux](https://talos.dev/) |
| CNI | [Cilium](https://cilium.io/) (L2/ARP mode, no kube-proxy) |
| GitOps | [ArgoCD](https://argoproj.github.io/argo-cd/) with Vault Plugin |
| Ingress | [Traefik](https://traefik.io/) |
| TLS | [cert-manager](https://cert-manager.io/) (Cloudflare DNS-01) |
| DNS | [External DNS](https://github.com/kubernetes-sigs/external-dns) (Cloudflare) |
| Storage | [OpenEBS LVM](https://openebs.io/) |
| Backup | [Volsync](https://github.com/backube/volsync) + Restic → local [Minio](https://min.io/) |
| Secrets | [Vault](https://www.vaultproject.io/) + [External Secrets](https://external-secrets.io/) |
| Monitoring | [Prometheus](https://prometheus.io/) + [Grafana](https://grafana.com/) + [Loki](https://grafana.com/oss/loki/) |

## Repository Layout

```
cluster/          # Single cluster definition
  patches/        # Talos machine config patches
  bootstrap/      # ArgoCD bootstrap apps + ApplicationSets
  system/         # Infrastructure components (storage, DNS, monitoring, etc.)
  argocd/        # ArgoCD install & config (also bootstrapped via extraManifests)
  cilium/        # Cilium CNI install (bootstrapped via extraManifests)
  apps/           # User applications (managed via ApplicationSet)
docs/             # This documentation
```

## Getting Started

See the [Getting Started](getting-started.md) guide for installation instructions.