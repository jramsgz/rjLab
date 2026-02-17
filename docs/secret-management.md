# Secret Management

To avoid storing secrets in the repository, we use Vault to store them (e.g., API keys, passwords, etc.). We then use the External Secrets operator to sync the secrets stored in Vault with the Kubernetes cluster.

## Vault

Vault is a tool for securely accessing secrets. A secret is anything that you want to tightly control access to, such as API keys, passwords, or certificates. Vault provides a unified interface to any secret while providing tight access control and recording a detailed audit log.

It is installed in the `vault` namespace using the ArgoCD application defined in `cluster/bootstrap/vault.yaml`.


### Initialize the Vault cluster

Every time the Vault pod restarts, it must be unsealed. We use a single unseal key (acceptable for a homelab).

```bash
kubectl exec -n vault vault-0 -- vault operator init \
    -key-shares=1 \
    -key-threshold=1 \
    -format=json > cluster-keys.json
```

You obtain the root token in the `cluster-keys.json` file. Ensure to keep it safe and **do not commit it to the repository** (or encrypt it if you do).

```bash
age -R ~/.ssh/id_ed25519.pub cluster-keys.json > cluster-keys.json.age # Encrypt the file
age -d -i ~/.ssh/id_ed25519 cluster-keys.json.age > cluster-keys.json # Decrypt the file
```

Now unseal Vault:

```bash
export VAULT_UNSEAL_KEY=$(jq -r ".unseal_keys_b64[]" cluster-keys.json)
kubectl exec -n vault -ti vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
```

You can also use the Taskfile shortcut: `task vault-unseal`.

If the Vault pod restarts, you will need to unseal it again.

### Enable the KV secret engine

Once Vault is ready, enable the KV v1 secret engine:

```bash
kubectl port-forward -n vault svc/vault 8200:8200 &
# Or use: task vault-port-forward

export VAULT_ADDR="http://localhost:8200"
export VAULT_TOKEN=$(jq -r ".root_token" cluster-keys.json)
vault secrets enable -path=kv -version=1 kv
```

To test the Vault cluster, we can write a secret in the KV secret engine.

```bash
vault kv put kv/foo my-value=s3cr3t
```

## External Secrets

External Secrets allows you to use secrets stored in external secret management systems like HashiCorp Vault. It is installed in the `external-secrets` namespace using the ArgoCD application in `cluster/system/external-secret/`.

*Note: This is actually the root token, which is not recommended for production use. In a production environment, you should create a dedicated token with the appropriate policies.*

***Create a secret with the Vault token***

This is already handled by the bootstrap process, but if needed manually:

```bash
kubectl create secret generic vault-token \
  --namespace external-secrets \
  --from-literal=token=$(jq -r ".root_token" cluster-keys.json)
```

***ClusterSecretStore***

The ClusterSecretStore is defined in `cluster/system/external-secret/cluster-store.yaml` and points to Vault.

### Test the External Secrets

***Create an External Secret that syncs the secret in Vault with the Kubernetes cluster***
```bash
cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: vault-example
  namespace: default
spec:
  refreshInterval: "15s"
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: example-sync # Name of the Secret in the target namespace
  data:
  - secretKey: foobar
    remoteRef:
      key: foo
      property: my-value
EOF
```

## ArgoCD Vault Plugin

For many applications, the ArgoCD Vault Plugin is used to inline Vault secrets into any Kubernetes manifest — not just Secrets but also ConfigMaps, Deployments, Ingresses, etc.


***Create a secret with the Vault token for the Vault Plugin***
```bash
kubectl create secret generic vault-credentials \
  --namespace argocd \
  --from-literal=AVP_AUTH_TYPE=token \
  --from-literal=AVP_KV_VERSION=1 \
  --from-literal=AVP_TYPE=vault \
  --from-literal=VAULT_ADDR=http://vault.vault.svc.cluster.local:8200 \
  --from-literal=VAULT_TOKEN=$(jq -r ".root_token" cluster-keys.json)
```

After creating this secret, upgrade ArgoCD to include the AVP sidecar containers:

```bash
task vault-avp
```

This applies the kustomization in `cluster/argocd/vault-argocd/`, which patches the `argocd-repo-server` Deployment with AVP sidecar containers and the CMP plugin ConfigMap.

Once everything is applied, you can use AVP placeholders in any manifest. For example, to hide the domain in an Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    cert-manager.io/cluster-issuer: cloudflare
spec:
  ingressClassName: traefik
  rules:
  - host: "my-app.<path:kv/cluster#domain>"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app
            port:
              number: 80
  tls:
  - hosts:
    - "my-app.<path:kv/cluster#domain>"
    secretName: my-app-tls
```

The `<path:kv/cluster#domain>` placeholder is replaced at sync time by the ArgoCD Vault Plugin with the value from Vault.

