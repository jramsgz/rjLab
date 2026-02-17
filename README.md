<p align="center">
    <br>
    <a href="#"><img src="https://readme-typing-svg.herokuapp.com?font=Fira+Code&pause=1000&center=true&vCenter=true&width=435&lines=Homelab+made+simple;Talos+go+brrrrr;GitOps+FTW" alt="Typing SVG" /></a>
</p>

<div align="center">

  [![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.32.3-blue?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
  [![Linux](https://img.shields.io/badge/Talos-v1.9.5-blue?style=for-the-badge&logo=linux&logoColor=white)](https://talos.dev/)

</div>

# HomeLab

<div align="center">

*Single-node homelab running on plain Talos Linux, managed entirely via GitOps.*

</div>

## Overview

This repository contains the complete infrastructure configuration for a single-node Kubernetes homelab. Everything is declared in Git and automatically reconciled by ArgoCD.

## Stack

- **OS**: [Talos Linux](https://www.talos.dev/) — immutable, API-driven Kubernetes OS
- **CNI**: [Cilium](https://cilium.io/) — eBPF-based networking with L2 LoadBalancer (ARP mode)
- **GitOps**: [ArgoCD](https://argoproj.github.io/argo-cd/) — continuous deployment from this repo
- **Ingress**: [Traefik](https://doc.traefik.io/traefik/) — ingress controller with middleware support
- **TLS**: [cert-manager](https://cert-manager.io/) — automated Let's Encrypt certificates via Cloudflare DNS-01
- **DNS**: [External DNS](https://github.com/kubernetes-sigs/external-dns) — automatic Cloudflare DNS record management
- **Secrets**: [HashiCorp Vault](https://www.vaultproject.io/) + [External Secrets](https://external-secrets.io/) + [ArgoCD Vault Plugin](https://argocd-vault-plugin.readthedocs.io/)
- **Storage**: [OpenEBS LVM](https://openebs.io/) — local persistent volumes on LVM volume group
- **Backup**: [Volsync](https://github.com/backube/volsync) + Restic → local [Minio](https://min.io/) (S3-compatible, on dedicated backup HDD)
- **Monitoring**: [Prometheus](https://prometheus.io/) + [Grafana](https://grafana.com/) + [Loki](https://grafana.com/oss/loki/)

## Repository Structure

```
.
├── cluster/              # Single cluster configuration
│   ├── apps/             # Application deployments (auto-discovered by ArgoCD)
│   ├── argocd/           # ArgoCD install & config
│   ├── bootstrap/        # ArgoCD bootstrap (Applications & ApplicationSets)
│   ├── cilium/           # Cilium CNI install
│   ├── patches/          # Talos machine config patches
│   └── system/           # System components (monitoring, storage, backup, etc.)
├── docs/                 # Documentation
├── Taskfile.yml          # Task runner for talosctl operations
└── .env.example          # Environment variables template
```

## Quick Start

1. **Get a Talos image** with the right extensions from [Talos Image Factory](https://factory.talos.dev/)
   - Include: `siderolabs/iscsi-tools`, `siderolabs/util-linux-tools`
2. **Boot the node** with the Talos image
3. **Configure** — copy `.env.example` to `.env` and fill in your values
4. **Deploy** — `task gen-config && task apply-config && task bootstrap && task kubeconfig`
5. **Wait** — ArgoCD will self-provision and deploy everything from this repo

See [docs/getting-started.md](docs/getting-started.md) for the full walkthrough.

## Hardware

- Single amd64 node
- 1x 1TB HDD — storage (OpenEBS LVM)
- 1x 1TB HDD — backups (Minio)

## Vault Secrets Reference

After cluster bootstrap, populate these Vault paths:

| Path | Keys | Used by |
|------|------|---------|
| `kv/cluster` | `domain`, `IP`, `email` | Ingress hosts, Cilium IP pool, cert-manager |
| `kv/cloudflare` | `dnsToken` | cert-manager, external-dns |
| `kv/grafana` | `user`, `pass`, `prometheus_user`, `prometheus_pass` | Grafana, Prometheus basic auth |
| `kv/kyoo` | `apikey`, `tmdb` | Kyoo media server |
| `kv/minio` | `rootUser`, `rootPassword` | Minio deployment |
| `kv/radarr` | `token` | Radarr |
| `kv/restic` | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `RESTIC_PASSWORD`, `RESTIC_REPOSITORY` | Volsync backups |
| `kv/sonarr` | `token` | Sonarr |

## Usage

```bash
# Install task runner: https://taskfile.dev/installation/

task gen-config        # Generate Talos machine config
task apply-config      # Apply config to node
task bootstrap         # Bootstrap cluster (first time only)
task kubeconfig        # Fetch kubeconfig
task health            # Check cluster health
task dashboard         # Open Talos dashboard
task vault-init        # Initialize Vault (first time only)
task vault-unseal      # Unseal Vault after restart
task upgrade-talos     # Upgrade Talos (pass version as arg)
task upgrade-k8s       # Upgrade Kubernetes (pass version as arg)
```

