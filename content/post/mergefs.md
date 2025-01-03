---
title: "使用mergerfs使用空闲硬盘"
date: 2024-06-25T11:44:42+08:00
tags: [ "fs", "samba", "nfs"]
categories: ['k8s-at-home']
draft: true
---

unraid 上面的unionfs很好的拯救空闲硬盘，在从unraid切换到truenas/pve后，就一直在想在非unraid上能不能实现类似的unionfs。
后来找到了mergerfs。
