---
title: "kvm 虚拟机去虚拟化"
date: 2023-02-01T18:56:05+08:00
slug: "hide-vm-info-from-guest"
tags: [ "linux", "kvm", "pve" ]
draft: false
---

使用promox ve做aio，启动windows主机当游戏机时，遇到游戏的虚拟机检测。

## 方法

在[promox ve forum](https://forum.proxmox.com/threads/hide-vm-from-guest.34905/)中找到了隐藏虚拟机标识的配置。

```
args: -cpu 'host,-hypervisor,+kvm_pv_unhalt,+kvm_pv_eoi,hv_spinlocks=0x1fff,hv_vapic,hv_time,hv_reset,hv_vpindex,hv_runtime,hv_relaxed,kvm=off,hv_vendor_id=intel'
```
