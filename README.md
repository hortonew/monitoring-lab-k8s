# Monitoring Lab: K8s

Successor to the [Docker Compose based monitoring lab](https://github.com/hortonew/monitoring-lab), this time in Kubernetes, hosted on my Proxmox server.

![Monitoring Lab on k8s](images/k8s-lab.png)

## ArgoCD

```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Bootstrap the cluster (or go in order 0-n)
kubectl apply -f argocd-apps/

# Make argocd accessible via MetalLB once it's set up
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Set up necessary secrets for cert-manager and external-dns
kubectl create secret generic cloudflare-api-token-secret --from-literal=api-token=<secret here> -n cert-manager
kubectl create secret generic cloudflare-api-token-secret --from-literal=api-token=<secret here> -n external-dns 
```

## Add container registry (harbor)

```sh
# https://github.com/goharbor/harbor-helm/blob/main/values.yaml
helm repo add harbor https://helm.goharbor.io
helm upgrade --install harbor harbor/harbor --namespace harbor --create-namespace -f k8s-configs/harbor-values.yml

# Create cert for nginx
openssl req -newkey rsa:2048 -nodes -keyout harbor-tls.key -x509 -days 365 -out harbor-tls.crt -subj "/CN=harbor.hortonew.com" -addext "subjectAltName=DNS:harbor.hortonew.com,IP:192.168.1.242"
kubectl create secret tls harbor-tls --cert=harbor-tls.crt --key=harbor-tls.key -n harbor
```

## Override DNS for pods to find harbor.hortonew.com

kubectl edit configmap coredns -n kube-system 
```sh
 13         ready
 14         hosts {
 15             192.168.1.242 harbor.hortonew.com
 16             fallthrough
 17         }
```

Then restart and test:

```sh
kubectl rollout restart deployment coredns -n kube-system
kubectl run busybox --image=busybox --rm -it --restart=Never -- nslookup harbor.hortonew.com
```

<!-- ## Gitlab

helm upgrade --install gitlab gitlab/gitlab --namespace gitlab --create-namespace -f k8s-configs/git-and-container-registry/gitlab-values.yml --timeout 600s -->