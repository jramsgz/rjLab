## Expose Applications with Cloudflare Tunnel (Optional)

> **Note**: This setup is not currently active. It's documented here for reference in case you want to expose services through Cloudflare Tunnel in the future instead of (or alongside) direct access.

Cloudflare Tunnel creates a secure outbound connection from the cluster to Cloudflare's edge, allowing you to expose services without opening inbound ports. This is free with any Cloudflare plan.

### Cloudflare

Install cloudflared, this is the client that will create the tunnel between the cluster and Cloudflare. You can still use the web interface to create the tunnel, but it's easier to manage it with the CLI.

```bash
brew install cloudflared
cloudflared login # Login to your Cloudflare account
```

Create a tunnel with the CLI, note the ID of the tunnel that will be used later.

```bash
cloudflared tunnel create my-cluster

# Save the tunnel ID and credentials JSON file
```

By creating the tunnel, you obtain a JSON file with the credentials that will be used to authenticate the tunnel with Cloudflare, keep it safe and **do not commit it to the repository** (or encrypt it if you do, even if it's still not recommended).

```bash
kubectl create namespace cloudflare
kubectl create secret generic tunnel-credentials \
  --from-file=credentials.json=/path/to/<TUNNEL_ID>.json \
  --namespace=cloudflare
```

Install the Cloudflare Tunnel Controller in the `cloudflare` namespace.

***`values.yaml`***
```yaml
cloudflare:
  tunnelName: "my-cluster"
  tunnelId: "<TUNNEL_ID>"
  secretName: "tunnel-credentials"
  ingress:
    - hostname: "*.yourdomain.com"
      service: "https://traefik.traefik.svc.cluster.local:443"
      originRequest:
        noTLSVerify: true

resources:
  limits:
    cpu: "100m"
    memory: "128Mi"
  requests:
    cpu: "100m"
    memory: "128Mi"

replicaCount: 1
```

```bash
helm repo add cloudflare https://cloudflare.github.io/helm-charts
helm repo update
helm upgrade --install cloudflare-tunnel cloudflare/cloudflare-tunnel \
  --namespace cloudflare \
  --values values.yaml
```

<details>
<summary>ArgoCD Application</summary>

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cloudflare-tunnel
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://cloudflare.github.io/helm-charts
    chart: cloudflare-tunnel
    targetRevision: 0.3.2
    helm:
      values: |
          cloudflare:
            tunnelName: "my-cluster"
            tunnelId: "<TUNNEL_ID>"
            secretName: "cloudflare-tunnel"
            ingress:
              - hostname: "*.yourdomain.com"
                service: "https://traefik.traefik.svc.cluster.local:443"
                originRequest:
                  noTLSVerify: true

          resources:
            limits:
              cpu: "100m"
              memory: "128Mi"
            requests:
              cpu: "100m"
              memory: "128Mi"

          replicaCount: 1
  destination:
    server: https://kubernetes.default.svc
    namespace: cloudflare
  syncPolicy:
    automated:
      prune: true
    syncOptions:
      - CreateNamespace=true
```

</details>

If you create a CNAME record in Cloudflare pointing to `<TUNNEL_ID>.cfargotunnel.com`, you can access the services on the cluster using the domain name.

### External DNS

External DNS is already configured in `cluster/system/external-dns/` and will automatically create DNS records in Cloudflare for each Ingress resource.

[Create an API key in Cloudflare that can edit your DNS records](https://dash.cloudflare.com/profile/api-tokens) of the domain (Zone) you configured in Cloudflare.

Store the token in Vault:

```bash
vault kv put kv/cloudflare dnsToken="your-cloudflare-api-token"
```

The ExternalSecret in `cluster/system/external-dns/external-secret.yaml` syncs this to a Kubernetes Secret automatically.


Now, you can add annotations to your Ingress resources to automatically create DNS records in Cloudflare.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    external-dns.alpha.kubernetes.io/hostname: my-app.yourdomain.com
  name: my-app
  namespace: default
spec:
  ingressClassName: traefik
  rules:
  - host: my-app.yourdomain.com
    http:
      paths:
      - backend:
          service:
            name: my-app
            port:
              number: 80
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - my-app.yourdomain.com
    secretName: my-app-tls
```

If using Cloudflare Tunnel, add the target annotation:
```yaml
external-dns.alpha.kubernetes.io/target: <TUNNEL_ID>.cfargotunnel.com
external-dns.alpha.kubernetes.io/cloudflare-proxied: "true"
```

The External DNS ArgoCD application is already defined in `cluster/system/external-dns/external-dns.yaml`.