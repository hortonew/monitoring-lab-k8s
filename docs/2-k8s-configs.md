# Configs that might help at some point


## Override DNS for pods to find harbor.lab.hortonew.com

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
kubectl run busybox --image=busybox --rm -it --restart=Never -- nslookup harbor.lab.hortonew.com
```
