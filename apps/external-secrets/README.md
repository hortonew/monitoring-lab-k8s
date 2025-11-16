# External Secrets Operator with HashiCorp Vault

This setup integrates External Secrets Operator (ESO) with HashiCorp Vault to automatically sync secrets from Vault into Kubernetes Secrets.

## Architecture

- **External Secrets Operator**: Watches `ExternalSecret` resources and syncs data from external secret stores
- **Vault Backend**: HashiCorp Vault using Kubernetes authentication
- **SecretStore**: Configuration for ESO to connect to Vault
- **ExternalSecret**: Defines which secrets to sync from Vault

## Prerequisites

1. Vault must be deployed, initialized, and unsealed (see `apps/vault/README.md`)
2. Vault must have Kubernetes authentication configured (see step 3 below)

## Quick Start

### 1. Deploy External Secrets Operator

```bash
kubectl apply -f argocd-apps/15-external-secrets.yml
```

Wait for the operator to be ready:
```bash
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=external-secrets -n external-secrets --timeout=300s
```

### 2. Apply Kubernetes Resources for Vault Authentication

Apply the ServiceAccount and RBAC resources needed for Vault authentication:

```bash
kubectl apply -f apps/vault/k8s/vault-kubernetes-auth-setup.yaml
```

This creates:
- ServiceAccount `external-secrets-vault` in the `external-secrets` namespace
- ClusterRole and ClusterRoleBinding for token review
- Service account token Secret

### 3. Configure Vault Kubernetes Authentication Manually

Exec into Vault and run the configuration commands:

```bash
# Exec into Vault
kubectl exec -it vault-0 -n vault -- sh

# Set your root token from Vault initialization
export VAULT_TOKEN='YOUR_ROOT_TOKEN_HERE'

# Enable Kubernetes auth method
vault auth enable kubernetes

# Configure the auth method
vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc.cluster.local" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Create a policy for external-secrets to read secrets
vault policy write external-secrets - <<EOF
path "secret/data/*" {
  capabilities = ["read", "list"]
}
path "secret/metadata/*" {
  capabilities = ["read", "list"]
}
EOF

# Create a role for external-secrets
# Note: The 'audience=vault' parameter is required for Vault 1.21+ and must match
# the 'audiences' field in the SecretStore/ClusterSecretStore configuration
vault write auth/kubernetes/role/external-secrets \
    bound_service_account_names=external-secrets-vault \
    bound_service_account_namespaces=external-secrets \
    policies=external-secrets \
    audience=vault \
    ttl=24h

# Enable KV v2 secrets engine (if not already enabled)
vault secrets enable -path=secret kv-v2

# Create a demo secret for testing
# vault kv put secret/demo/credentials \
#     username=admin \
#     password=changeme123 \
#     api-key=demo-api-key-12345

# Exit the pod
exit
```

**Note**: This manual approach is more secure as it doesn't require storing the Vault root token in Kubernetes.

### 4. Verify the Setup

Check that the ClusterSecretStore is ready:
```bash
kubectl get clustersecretstore
kubectl describe clustersecretstore vault-backend
```

The ClusterSecretStore should show `STATUS: Valid` and `READY: True`.

Check that the ExternalSecrets are syncing:
```bash
kubectl get externalsecret -n external-secrets
kubectl describe externalsecret demo-credentials -n external-secrets
```

Verify the synced Kubernetes Secrets were created:
```bash
kubectl get secrets -n external-secrets | grep demo-credentials
kubectl get secret demo-credentials -n external-secrets -o yaml
```

If the ClusterSecretStore shows as "Ready: True" and the ExternalSecrets show "STATUS: SecretSynced", your configuration is complete!

### 5. Test the Secret Data

```bash
# View the synced secret
kubectl get secret demo-credentials -n external-secrets -o jsonpath='{.data.username}' | base64 -d
# Output: admin

kubectl get secret demo-credentials -n external-secrets -o jsonpath='{.data.password}' | base64 -d
# Output: changeme123
```

## How It Works

### 1. SecretStore Configuration

The `SecretStore` or `ClusterSecretStore` resource tells ESO how to connect to Vault:

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "http://vault.vault.svc.cluster.local:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"
          serviceAccountRef:
            name: "external-secrets-vault"
            namespace: "external-secrets"
            # audiences must match the audience configured in Vault role
            # Required for Vault 1.21+ compatibility
            audiences:
            - "vault"
```

**Note**: Use `ClusterSecretStore` to share the Vault connection across all namespaces, or `SecretStore` for namespace-specific configurations.

### 2. ExternalSecret Resources

The `ExternalSecret` resources define which secrets to sync:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: demo-credentials
  namespace: external-secrets
spec:
  refreshInterval: 1m
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: demo-credentials
  data:
  - secretKey: username
    remoteRef:
      key: demo/credentials
      property: username
```

## Managing Secrets in Vault

### Add a New Secret to Vault

```bash
export VAULT_ADDR=http://vault.lab.horton.com:8200
export VAULT_TOKEN='YOUR_ROOT_TOKEN'

# Add a new secret
vault kv put secret/myapp/database \
  host=postgres.example.com \
  username=dbuser \
  password=secretpass
```

### Create an ExternalSecret for Your App

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: myapp-database
  namespace: myapp
spec:
  refreshInterval: 5m
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore  # Use ClusterSecretStore for cross-namespace
  target:
    name: database-credentials
  data:
  - secretKey: DB_HOST
    remoteRef:
      key: myapp/database
      property: host
  - secretKey: DB_USERNAME
    remoteRef:
      key: myapp/database
      property: username
  - secretKey: DB_PASSWORD
    remoteRef:
      key: myapp/database
      property: password
```

## Examples Included

1. **demo-credentials**: Individual field mapping from Vault
2. **demo-credentials-all**: Pull entire secret (all key-value pairs)
3. **demo-credentials-templated**: Transform secret data using templates

## Troubleshooting

### Check ESO Logs

```bash
kubectl logs -n external-secrets -l app.kubernetes.io/name=external-secrets --tail=50 -f
```

### Check ExternalSecret Status

```bash
kubectl describe externalsecret demo-credentials -n external-secrets
```

Look for events and the status conditions.

### Test Vault Authentication

```bash
# Get a token from Kubernetes
kubectl exec -n external-secrets deployment/external-secrets -it -- sh
# In the pod:
export VAULT_ADDR=http://vault.vault.svc.cluster.local:8200
export SA_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

# Try to authenticate
curl -sf --request POST \
  --data '{"jwt":"'"$SA_TOKEN"'","role":"external-secrets"}' \
  $VAULT_ADDR/v1/auth/kubernetes/login | jq
```

### Common Issues

1. **SecretStore not ready**: Check that Vault is accessible and Kubernetes auth is configured
2. **ExternalSecret sync failed**: Check the SecretStore configuration and Vault policies
3. **Permission denied**: Verify the Vault role and policy are correctly configured

## Security Considerations

- The External Secrets Operator only needs read access to secrets in Vault
- Kubernetes Secrets are created in the namespace where the ExternalSecret resource exists
- Use RBAC to control who can create ExternalSecret resources
- Consider using ClusterSecretStore for shared Vault connections
- Rotate Vault tokens regularly

## References

- [External Secrets Operator Documentation](https://external-secrets.io/)
- [Vault Kubernetes Auth](https://developer.hashicorp.com/vault/docs/auth/kubernetes)
- [ESO Vault Provider](https://external-secrets.io/latest/provider/hashicorp-vault/)
