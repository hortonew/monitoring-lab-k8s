# Monitoring Lab: K8s

Successor to the [Docker Compose based monitoring lab](https://github.com/hortonew/monitoring-lab), this time in Kubernetes.

![Monitoring Lab on k8s](images/k8s-lab.png)

## Assumptions

The hosts in the inventory are reachable and you can authenticate with your SSH key to the ubuntu user on each machine.

1. Create 5 VMs with ubuntu user
2. Add SSH public key to ~/.ssh/authorized_keys of ubuntu user
3. Make sure all hosts are reachable via hostname from ansible machine.

## 🤝 Dependencies

- ansible
- 5 servers (VMs): k8s1, k8s2, k8s3, k8skw1, k8skw2

## ⏩ Quickstart

Run the ansible playbook:

```sh
ansible-playbook configure-k8s-cluster.yml
```

## Optional

If you need to target specific tags or hosts, here are some examples:

```sh
# Install dependencies on all hosts
ansible-playbook configure-k8s-cluster.yml -t setup

# Other examples
ansible-playbook configure-k8s-cluster.yml -t hostname --limit control_nodes
ansible-playbook configure-k8s-cluster.yml -t k8s --limit primary_control_node,secondary_control_nodes
```

Update ubuntu

```sh
ansible-playbook update-ubuntu.yml
```

Get k8s version

```sh
ansible-playbook get-k8s-versions.yml
```
