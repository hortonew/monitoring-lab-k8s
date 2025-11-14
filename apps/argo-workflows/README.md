# Argo Workflows & Argo Events

This directory contains the configuration for deploying Argo Workflows and Argo Events to the Kubernetes cluster.

## Overview

**Argo Workflows** is an open-source container-native workflow engine for orchestrating parallel jobs on Kubernetes. It enables you to define workflows where each step is a container, making it ideal for CI/CD, data processing, and machine learning pipelines.

**Argo Events** is an event-driven workflow automation framework that triggers Argo Workflows based on events from various sources (webhooks, message queues, etc.).

## Architecture

This deployment consists of three ArgoCD applications:

1. **argo-workflows** (`12-argo-workflows.yml`) - Deploys the core Argo Workflows components via Helm
2. **argo-events** (`13-argo-events.yml`) - Deploys Argo Events via Helm
3. **argo-workflows-config** (`14-argo-workflows-config.yml`) - Deploys custom configurations (EventBus, Sensors, WorkflowTemplates)

## Components Deployed

### From Helm Charts:
- Argo Workflows Server (UI and API)
- Workflow Controller
- Argo Events Controller

### From Manifests (`k8s/` directory):
- **EventBus** - NATS-based event bus for Argo Events
- **EventSource** - Webhook event source
- **Sensors** - Event listeners that trigger workflows
- **WorkflowTemplates** - Reusable workflow definitions
- **Services** - Kubernetes services for webhook access
- **RBAC** - Service accounts and permissions

## Directory Structure

```
apps/argo-workflows/
├── helm/
│   ├── argo-workflows-values.yaml  # Helm values for Argo Workflows
│   └── argo-events-values.yaml     # Helm values for Argo Events
└── k8s/
    ├── eventbus.yaml                          # NATS event bus configuration
    ├── webhook-eventsource.yaml               # Webhook event source
    ├── webhook-basic-sensor.yaml              # Basic webhook sensor
    ├── webhook-kubectl-sensor.yaml            # Kubectl workflow sensor
    ├── webhook-service.yaml                   # Webhook service
    ├── webhook-event-binding.yaml             # Event binding configuration
    ├── webhook-workflow-template.yaml         # Basic workflow template
    ├── kubectl-failed-pods-workflow-template.yaml  # Failed pods check workflow
    └── argo-events-rbac.yaml                  # RBAC for Argo Events
```

## Deployment

### Prerequisites

- ArgoCD installed and configured
- kubectl access to the cluster
- (Optional) Prometheus for metrics collection

### Deploy with ArgoCD

The applications are deployed automatically by ArgoCD when the manifests are committed:

```bash
# Apply all Argo-related applications
kubectl apply -f argocd-apps/12-argo-workflows.yml
kubectl apply -f argocd-apps/13-argo-events.yml
kubectl apply -f argocd-apps/14-argo-workflows-config.yml

# Or apply them in order as part of the full stack
kubectl apply -f argocd-apps/
```

### Manual Deployment (Alternative)

If you prefer to deploy without ArgoCD:

```bash
# Create namespace
kubectl create namespace argo

# Install Argo Workflows via Helm
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install argo-workflows argo/argo-workflows \
    --namespace argo \
    --values apps/argo-workflows/helm/argo-workflows-values.yaml \
    --wait

# Install Argo Events via Helm
helm upgrade --install argo-events argo/argo-events \
    --namespace argo \
    --values apps/argo-workflows/helm/argo-events-values.yaml \
    --wait

# Apply custom configurations
kubectl apply -f apps/argo-workflows/k8s/
```

## Accessing Argo Workflows UI

### Port Forward (Development)

```bash
kubectl port-forward -n argo svc/argo-workflows-server 2746:2746
```

Then open: http://localhost:2746

### LoadBalancer (Production)

Edit `argocd-apps/12-argo-workflows.yml` to enable LoadBalancer service:

```yaml
server:
  service:
    type: LoadBalancer
    annotations:
      external-dns.alpha.kubernetes.io/hostname: argo-workflows.lab.hortonew.com
```

### Ingress (Production - Alternative)

Edit the Helm values to enable ingress:

```yaml
server:
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - argo-workflows.lab.hortonew.com
    tls:
      - secretName: argo-workflows-tls
        hosts:
          - argo-workflows.lab.hortonew.com
```

## Using Argo Workflows

### Submit a Workflow via CLI

```bash
# Install argo CLI
brew install argo  # macOS
# or download from https://github.com/argoproj/argo-workflows/releases

# Submit a workflow
argo submit -n argo --watch apps/argo-workflows/k8s/webhook-workflow-template.yaml

# List workflows
argo list -n argo

# Get workflow details
argo get -n argo <workflow-name>

# View workflow logs
argo logs -n argo <workflow-name>
```

### Trigger via Webhook

The webhook event source is available at port 12000:

```bash
# Port forward the webhook service
kubectl port-forward -n argo svc/webhook-eventsource-svc 12000:12000

# Trigger a workflow
curl -X POST http://localhost:12000/webhook \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Test workflow",
    "source": "curl",
    "workflow": "basic"
  }'
```

### Check Workflow Status

```bash
# List all workflows
kubectl get workflows -n argo

# Watch workflows in real-time
kubectl get workflows -n argo -w

# Describe a workflow
kubectl describe workflow -n argo <workflow-name>
```

## Available Workflow Templates

### 1. Basic Webhook Workflow
**Template:** `webhook-workflow-template.yaml`  
**Trigger:** Webhook with `"workflow": "basic"`  
**Description:** Simple workflow that echoes webhook payload

### 2. Failed Pods Check Workflow
**Template:** `kubectl-failed-pods-workflow-template.yaml`  
**Trigger:** Webhook with `"workflow": "kubectl-failed-pods"`  
**Description:** Checks cluster for failed pods and reports them

Example trigger:
```bash
curl -X POST http://localhost:12000/webhook \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Check for failed pods",
    "workflow": "kubectl-failed-pods"
  }'
```

## Integration with Prometheus

The Workflow Controller exports metrics on port 9090 that Prometheus can scrape:

```yaml
# ServiceMonitor is created automatically by the Helm chart
apiVersion: v1
kind: Service
metadata:
  name: argo-workflows-controller-metrics
  namespace: argo
spec:
  ports:
    - name: metrics
      port: 9090
      targetPort: 9090
```

Available metrics include:
- `argo_workflows_count` - Total number of workflows
- `argo_workflow_condition` - Workflow conditions
- `argo_workflows_queue_depth` - Workflow queue depth
- And many more...

## Configuration

### Modify Workflow Namespaces

By default, workflows can run in `argo` and `default` namespaces. To add more:

Edit `argocd-apps/12-argo-workflows.yml`:
```yaml
controller:
  workflowNamespaces:
    - argo
    - default
    - my-namespace
```

### Adjust Resource Limits

Modify the resources section in `argocd-apps/12-argo-workflows.yml` or the Helm values files.

### Enable Artifact Storage

To enable artifact storage with MinIO:

1. Uncomment the artifactRepository section in Helm values
2. Create the required secrets:

```bash
kubectl create secret generic argo-artifacts-s3 \
  --from-literal=accessKey=minioadmin \
  --from-literal=secretKey=minioadmin \
  -n argo
```

### Enable Workflow Archive

To persist workflow history to a database, uncomment the persistence section in Helm values and configure your PostgreSQL or MySQL connection.

## Troubleshooting

### Check ArgoCD Application Status
```bash
kubectl get applications -n argocd | grep argo
```

### Check Pod Status
```bash
kubectl get pods -n argo
```

### View Logs
```bash
# Workflow controller logs
kubectl logs -n argo -l app.kubernetes.io/name=argo-workflows-workflow-controller

# Argo server logs
kubectl logs -n argo -l app.kubernetes.io/name=argo-workflows-server

# Event source logs
kubectl logs -n argo -l eventsource-name=webhook-eventsource
```

### Common Issues

**Workflows stuck in Pending:**
- Check resource limits in the namespace
- Verify service account permissions
- Check the workflow controller logs

**Webhook not responding:**
- Ensure the event source pod is running
- Verify the service is created
- Check port forwarding is active

**Events not triggering workflows:**
- Verify EventBus is running (`kubectl get eventbus -n argo`)
- Check sensor logs
- Ensure event source is receiving events

## Security Considerations

⚠️ **Important:** The current configuration has `--auth-mode=server` and `--secure=false` for ease of use. For production:

1. Enable proper authentication:
   ```yaml
   server:
     extraArgs:
       - --auth-mode=sso  # or 'client'
       - --secure=true
   ```

2. Configure SSO or client certificates
3. Enable RBAC for workflow execution
4. Use network policies to restrict access
5. Enable TLS for ingress

## Resources

- [Argo Workflows Documentation](https://argo-workflows.readthedocs.io/)
- [Argo Events Documentation](https://argoproj.github.io/argo-events/)
- [Workflow Examples](https://github.com/argoproj/argo-workflows/tree/master/examples)
- [Event Source Examples](https://github.com/argoproj/argo-events/tree/master/examples)

## Next Steps

1. Explore the workflow templates in the UI
2. Create custom WorkflowTemplates for your use cases
3. Set up additional event sources (GitHub, GitLab, SNS, etc.)
4. Configure artifact storage for workflow outputs
5. Integrate with CI/CD pipelines
6. Set up monitoring and alerting with Prometheus/Grafana
