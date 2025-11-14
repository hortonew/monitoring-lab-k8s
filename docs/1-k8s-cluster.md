# Kubernetes

Create a kubernetes cluster:

```sh
ansible-playbook fix-machine-ids.yml
ansible-playbook configure-k8s-cluster.yml
```

## Pull down config, merge with existing configs, overwrite old

Ansible will pull down the config, just make sure you set the following manually, or in your .rc file

```sh
export KUBECONFIG=~/.kube/config:/tmp/proxmox-kubeconfig
```

## Configure strictArp on kubeproxy ipvs settings

[https://metallb.universe.tf/installation/#preparation](https://metallb.universe.tf/installation/#preparation)


Upgrade kubernetes:

```sh
ansible-playbook upgrade-k8s-version.yml -e target_k8s_version=1.33
```

Optional:

If you need to target specific tags or hosts, here are some examples:

```sh
# Install dependencies on all hosts
ansible-playbook configure-k8s-cluster.yml -t setup

# Other examples
ansible-playbook configure-k8s-cluster.yml -t hostname --limit control_nodes
ansible-playbook configure-k8s-cluster.yml -t k8s --limit primary_control_node,secondary_control_nodes
```

Update ubuntu:

```sh
ansible-playbook update-ubuntu.yml
```

Get k8s version:

```sh
ansible-playbook get-k8s-versions.yml
```

Uninstall everything to start from scratch:

```sh
ansible-playbook wipe-k8s-cluster.yml
```