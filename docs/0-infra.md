# Infrastructure

## Assumptions

The hosts in the inventory are reachable and you can authenticate with your SSH key to the ubuntu user on each machine.

1. Create 5 VMs with ubuntu user
2. Add SSH public key to ~/.ssh/authorized_keys of ubuntu user
3. Make sure all hosts are reachable via hostname from ansible machine.

## 🤝 Dependencies

- ansible
- 5 servers (VMs): k8s1, k8s2, k8s3, k8skw1, k8skw2

## Set up 5 identical ubuntu instances

Names: k8s1, k8s2, k8s3, k8skw1, k8skw2

## Expand disk if running low

```sh
# resize in proxmox, then:
sudo growpart /dev/sda 2
sudo resize2fs /dev/sda2
df -h
```

## Add disk to worker

1. Add disk in proxmox
2. Configure disk
```sh
sudo su -
lsblk # find new disk (e.g. sdb or sdc)
fdisk /dev/sdc
# n, p, <enter>, <enter>, <enter>, w
# aka: new, partition, use all space, write
mkfs.ext4 /dev/sdc1
mkdir /mnt/longhorn-disk1
mount /dev/sdc1 /mnt/longhorn-disk1

blkid /dev/sdc1
# copy uuid
vim /etc/fstab
# UUID=<uuid> /mnt/longhorn-disk1 ext4 defaults 0 0
```
3. Go to longhorn UI -> node -> edit disk -> add disk -> /mnt/longhorn-disk1