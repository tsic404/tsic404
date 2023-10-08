---
title: "Helm"
date: 2023-09-26T15:41:41+08:00
draft: true
---
# k3s 杂记

## 使用helm安装nfs-subdir-external-provisioner

```shell
helm -n storage-provisioner --create-namespace install nfs-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=192.168.18.25 \
    --set nfs.path=/mnt/data/data \
    --set storageClass.name=truenas \
    --set storageClass.archiveOnDelete=false \
    --set storageClass.provisionerName=k8s-sigs.io/second-nfs-subdir-external-provisioner \
    --set storageClass.pathPattern=\${.PVC.namespace}/\${.PVC.annotations.nfs.io/storage-path} \
    --set storageClass.onDelete=retain
```
