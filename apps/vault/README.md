# HashiCorp Vault with Raft Storage and Auto-Unseal

This setup deploys HashiCorp Vault in High Availability mode with:
- **Raft Integrated Storage** (no external storage dependency)
- **Shamir Seal** with automated unsealing via Kubernetes CronJob
- **3 Vault replicas** for high availability (can tolerate 1 node failure)
- **Prometheus monitoring** integration
- **External DNS** integration for automatic DNS records

## Architecture

The deployment uses:
1. **Vault cluster**: 3 replicas with Raft storage for HA
2. **Auto-unsealer CronJob**: Runs every 2 minutes to automatically unseal pods using keys stored in Kubernetes secrets

When Vault pods restart, they start in a sealed state. The CronJob detects sealed pods and automatically unseals them using the unseal keys stored in a Kubernetes secret. This provides automatic recovery without compromising security.

## Prerequisites

- Kubernetes cluster with ArgoCD installed
- MetalLB for LoadBalancer services
- External-DNS for automatic DNS record creation
- Prometheus for monitoring (optional)
- Storage class for persistent volumes (default storage class will be used)

## Quick Start

### 1. Deploy via ArgoCD

```bash
kubectl apply -f argocd-apps/14-vault.yml
```

This will deploy the Vault cluster and auto-unsealer CronJob.

### 2. Wait for Vault pods to be ready

```bash
kubectl wait --for=condition=ready pod vault-0 -n vault --timeout=300s
```

### 3. Initialize the Vault cluster (first time only)

```bash
# Initialize Vault with 3 key shares, requiring 2 to unseal
kubectl exec vault-0 -n vault -- vault operator init -key-shares=3 -key-threshold=2
```

**Important**: Save the output! You'll get:
- 3 Unseal Keys (need any 2 to unseal)
- Initial Root Token (for administration)

### 4. Manually unseal the first time

```bash
# Unseal vault-0 (need to provide 2 different keys)
kubectl exec vault-0 -n vault -- vault operator unseal <UNSEAL_KEY_1>
kubectl exec vault-0 -n vault -- vault operator unseal <UNSEAL_KEY_2>

# Verify it's unsealed
kubectl exec vault-0 -n vault -- vault status
```

### 5. Create the auto-unseal secret

```bash
# Store your unseal keys in a Kubernetes secret
kubectl create secret generic vault-unseal-keys -n vault \
  --from-literal=key1='<UNSEAL_KEY_1>' \
  --from-literal=key2='<UNSEAL_KEY_2>' \
  --from-literal=key3='<UNSEAL_KEY_3>'
```

From now on, the CronJob will automatically unseal any sealed Vault pods every 2 minutes!

### 6. Join the other nodes to the Raft cluster

```bash
# Join vault-1 to the cluster
kubectl exec vault-1 -n vault -- vault operator raft join http://vault-0.vault-internal:8200

# Join vault-2 to the cluster
kubectl exec vault-2 -n vault -- vault operator raft join http://vault-0.vault-internal:8200
```

The CronJob will automatically unseal vault-1 and vault-2 within 2 minutes.

### 7. Verify the Raft cluster

```bash
# Use your root token from step 3
export ROOT_TOKEN='YOUR_ROOT_TOKEN_HERE'

kubectl exec vault-0 -n vault -- sh -c "export VAULT_TOKEN=$ROOT_TOKEN && vault operator raft list-peers"
```

You should see all 3 nodes listed.

## Access Vault

### Via LoadBalancer

Get the external IP:
```bash
kubectl get svc -n vault vault-ui
```

Access the UI at: `http://<EXTERNAL-IP>:8200`

### Via DNS (if External-DNS is configured)

Access the UI at: `http://vault.lab.hortonew.com:8200`

## Port Forward (for local access)

```bash
kubectl port-forward -n vault vault-0 8200:8200
```

Then access: `http://localhost:8200`

## Common Operations

### Check Vault Status

```bash
kubectl exec -it vault-0 -n vault -- vault status
```

### View Raft Peers

```bash
kubectl exec -it vault-0 -n vault -- vault operator raft list-peers
```

### Login to Vault

```bash
# Set the address
export VAULT_ADDR='http://vault.lab.hortonew.com:8200'

# Login with root token
vault login

# Or with AppRole, LDAP, etc. after configuration
vault login -method=userpass username=myuser
```

### Take a Raft Snapshot (Backup)

```bash
kubectl exec -it vault-0 -n vault -- vault operator raft snapshot save /tmp/vault-snapshot.snap
kubectl cp vault/vault-0:/tmp/vault-snapshot.snap ./vault-snapshot-$(date +%Y%m%d).snap
```

### Restore from Raft Snapshot

```bash
kubectl cp ./vault-snapshot.snap vault/vault-0:/tmp/vault-snapshot.snap
kubectl exec -it vault-0 -n vault -- vault operator raft snapshot restore /tmp/vault-snapshot.snap
```

## Monitoring

Prometheus ServiceMonitors are automatically created for Vault. Metrics are available at:
- `http://vault:8200/v1/sys/metrics?format=prometheus`

## Troubleshooting

### Vault pods are sealed after restart

This is normal! The auto-unsealer CronJob runs every 2 minutes and will automatically unseal them.

To check the CronJob:
```bash
kubectl get cronjob vault-auto-unsealer -n vault
kubectl get jobs -n vault
```

To manually trigger the unsealer:
```bash
kubectl create job --from=cronjob/vault-auto-unsealer vault-manual-unseal -n vault
```

### Check if unseal keys secret exists

```bash
kubectl get secret vault-unseal-keys -n vault
```

If missing, recreate it with your unseal keys from initialization.

### Verify auto-unsealer is working

Check the CronJob logs:
```bash
kubectl logs -n vault -l job-name=$(kubectl get jobs -n vault -o name | grep vault-auto | head -1 | cut -d/ -f2)
```

### Raft cluster is not forming

1. Check that all pods can communicate:
   ```bash
   kubectl exec vault-0 -n vault -- nslookup vault-internal
   ```

2. Verify the Raft configuration (need root token):
   ```bash
   kubectl exec vault-0 -n vault -- sh -c "export VAULT_TOKEN=<your-root-token> && vault operator raft list-peers"
   ```

### Manual unseal if needed

If you need to manually unseal (e.g., before the CronJob runs):
```bash
kubectl exec vault-0 -n vault -- vault operator unseal <KEY_1>
kubectl exec vault-0 -n vault -- vault operator unseal <KEY_2>
```

This should not happen with auto-unseal. If it does:
1. Check the Transit unsealer is still running
2. Verify network connectivity between Vault and unsealer
3. Check the Transit key still exists

## Security Considerations

1. **Unsealer Root Token**: The unsealer runs with a static root token (`root`) for simplicity. In production, consider:
   - Using a more secure token
   - Implementing token rotation
## Security Considerations

1. **Unseal Keys Storage**: The unseal keys are stored in a Kubernetes secret
   - This provides convenience with reasonable security for a lab
   - For production, consider using a hardware security module (HSM)
   - Or use cloud KMS (AWS KMS, Azure Key Vault, GCP Cloud KMS)
   - The auto-unsealer runs with limited permissions

2. **TLS**: This setup uses HTTP for simplicity. For production:
   - Enable TLS in the Vault configuration
   - Use cert-manager to manage certificates

3. **Root Token**: Store the main Vault's root token securely:
   - Use it only for initial setup
   - Create admin policies and tokens for regular use
   - Consider revoking the root token after setup

4. **Backups**: Regularly backup Raft snapshots:
   - Automate snapshot creation
   - Store snapshots in a secure, off-cluster location
   - Test restore procedures

5. **Auto-unsealer CronJob**: The CronJob has read-only access to the unseal keys secret and can execute commands in Vault pods. This is necessary for auto-unsealing but should be monitored.

## Upgrading

To upgrade Vault:

1. Update the `targetRevision` in `argocd-apps/14-vault.yml`
2. ArgoCD will automatically roll out the upgrade
3. Raft will handle the rolling update gracefully with no downtime

## Uninstall

```bash
kubectl delete -f argocd-apps/14-vault.yml
```

Note: This will not delete PVCs. To fully clean up:

```bash
kubectl delete pvc -n vault -l app.kubernetes.io/name=vault
kubectl delete namespace vault
```

## Files Structure

```
apps/vault/
├── k8s/
│   ├── vault-auto-unsealer-job.yaml  # CronJob to auto-unseal pods
│   ├── vault-loadbalancer.yaml       # LoadBalancer service (optional)
│   └── servicemonitor.yaml           # Prometheus monitoring
└── README.md                          # This file

argocd-apps/
└── 14-vault.yml                       # ArgoCD applications
```
│   ├── vault-values.yaml          # Main Vault configuration
│   └── vault-unsealer-values.yaml # Unsealer configuration
├── k8s/
│   ├── vault-loadbalancer.yaml    # LoadBalancer service
│   └── servicemonitor.yaml        # Prometheus monitoring
└── README.md                       # This file

argocd-apps/
└── 14-vault.yml                    # ArgoCD application manifest
```

## References

- [HashiCorp Vault Documentation](https://www.vaultproject.io/docs)
- [Vault Helm Chart](https://github.com/hashicorp/vault-helm)
- [Raft Storage Backend](https://www.vaultproject.io/docs/configuration/storage/raft)
- [Transit Auto-Unseal](https://www.vaultproject.io/docs/concepts/seal#transit)
