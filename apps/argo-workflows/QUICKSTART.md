# Quick Start: Deploying Argo Workflows

## Deploy via ArgoCD (Recommended)

```bash
# Deploy all three Argo applications
kubectl apply -f argocd-apps/12-argo-workflows.yml
kubectl apply -f argocd-apps/13-argo-events.yml
kubectl apply -f argocd-apps/14-argo-workflows-config.yml

# Watch the deployment
watch kubectl get applications -n argocd

# Check pods
kubectl get pods -n argo
```

## Access the UI

```bash
# Port forward
kubectl port-forward -n argo svc/argo-workflows-server 2746:2746

# Open in browser
open http://localhost:2746
```

## Test with a Workflow

```bash
# Install argo CLI
brew install argo

# Submit test workflow
argo submit -n argo --watch https://raw.githubusercontent.com/argoproj/argo-workflows/master/examples/hello-world.yaml

# Or use the webhook
kubectl port-forward -n argo svc/webhook-eventsource-svc 12000:12000

curl -X POST http://localhost:12000/webhook \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello from webhook", "workflow": "basic"}'
```

## Verify Everything Works

```bash
# Check all Argo components
kubectl get all -n argo

# Check EventBus
kubectl get eventbus -n argo

# Check EventSources
kubectl get eventsources -n argo

# Check Sensors
kubectl get sensors -n argo

# List workflows
argo list -n argo
```

## Troubleshooting

```bash
# View logs
kubectl logs -n argo -l app.kubernetes.io/name=argo-workflows-workflow-controller
kubectl logs -n argo -l app.kubernetes.io/name=argo-workflows-server
kubectl logs -n argo -l eventsource-name=webhook-eventsource

# Describe resources
kubectl describe application argo-workflows -n argocd
```
