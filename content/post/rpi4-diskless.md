---
title: "Rpi4 Diskless"
date: 2023-07-24T10:52:40+08:00
tags: [ "pxe", "rpi4"]
categories: ['k8s-at-home']
draft: false
---

使用tftp + nfssroot/iscsi 实现树莓派无盘启动。

# 树莓派使用网络启动

pi  通过 tftp 和 dhcp 拉取 boot 分区的内容。然后把拉取的 boot 分区中的 initramfs 加载到内存，通过 内核启动参数（nfsroot，iscsi）和iniramfs 完成网络启动过程。

其他比如 rock5b 不支持 tftp, 但是可以用后面 initramfs 和内核参数。所以理论上可以做到本地 sd 卡只存放 boot 分区，然后通过网络启动 rootfs。

## 一、 tftp-hpa

1. 安装 tftp-hpa

`sudo apt install tftp-hpa`

2. 修改配置文件

`sudo vim /etc/default/tftpd-hpa`

修改一下配置

```
# /etc/default/tftpd-hpa

TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/data/pxe/"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure"
```

3. 重启 tftp-hpa 服务

`sudo systemctl restart tftp-hpa`

4. 在 TFTP_DIRECTORY 放置对应的网络启动文件。

将 pi img 中 boot 分区相关的文件放到对应的目录下，下文修改 boot 相关内容时，需要同步 tftp server中。

```
root@rock5b:~# ls /data/pxe/
rapi4-1_boot  rapi4-2_boot  rapi4-3_boot rpi4-1-root.img  rpi4-2-root.img  rpi4-3-root.img
```

## 二、 nfsroot

相关链接： https://docs.kernel.org/admin-guide/nfs/nfsroot.html
    
1. 安装nfs服务

`sudo apt install nfs-kernel-server`

2. 生成nfs路径

`mkdir /data/pxe/nfsroot1`

`vim /etc/exportfs`

加入以下内容
```
# rootfs
/data/pxe/nfsroot1    192.168.28.1/24(rw,no_root_squash,async,insecure)

# boot
/data/pxe/ 192.168.28.1/24(rw,no_root_squash,async,insecure)
```

然后在 pi 上尝试挂载 nfs，并将 rootfs 同步到nfs中。

```shell
mount 192.168.18.14:/data/pxe/nfsroot1 /mnt
rsync -avz --exclude /mnt --exclude /sys --exclude /proc --exclude /run --exclude /boot / /mnt
```

修改 nfsroot 中的 fstab(/mnt/etc/fstab)，将一些 tmp 和 log 挂载为 tmpfs 节省网络资源。
```
proc            /proc           proc    defaults          0       0
192.168.18.14:/data/pxe/rapi4-1_boot/ /boot nfs defaults,vers=4.1,proto=tcp 0 0
tmpfs   /tmp         tmpfs   rw,nodev,nosuid,size=2G,mode=777          0  0
tmpfs   /var/log         tmpfs   rw,nodev,nosuid,size=2G,mode=777          0  0
tmpfs   /var/tmp         tmpfs   rw,nodev,nosuid,size=2G,mode=777          0  0
```

3. 修改pi内核启动参数

生成initramfs。
```shell
update-initramfs -u -k `uname -r`
```
然后修改boot/config.txt, 加载initramfs。

```shell
vim /boot/config.txt
```

加入以下内容，注意修改成对应的 iniramfs 文件。
```
initramfs initrd.img-6.1.21-v8+ followkernel
```

修改内核参数 /boot/cmdline.txt
```
console=serial0,115200 console=tty1 ip=192.168.18.15::192.168.18.1:255.255.255.0:rpi4-1:eth0:off root=/dev/nfs nfsroot=192.168.18.14:/data/pxe/nfsroot1,tcp,rw rootfstype=nfs rootwait elevator=deadline cgroup_memory=1 cgroup_enable=memory cgroup_enable=cpuset
```

![](https://s2.loli.net/2023/07/24/w3SYbkex57HVoID.png)

注意将修改后的 boot 同步到 tftp server pxe 上。

## 三、 bootcmd

1. 查看当前bootloader配置
```shell
root@rpi4-1:~# vcgencmd bootloader_config
[all]
BOOT_UART=0
```
2. 提取配置文件
```
cp /lib/firmware/raspberrypi/bootloader/stable/pieeprom-2023-05-11.bin pieeprom.bin
rpi-eeprom-config pieeprom.bin > bootconf.txt
```
3. 设置配置文件
```
# bootconf.txt

[all]
BOOT_UART=0
WAKE_ON_GPIO=1
POWER_OFF_ON_HALT=0
DHCP_TIMEOUT=4000
DHCP_REQ_TIMEOUT=4000
TFTP_FILE_TIMEOUT=30000
TFTP_IP=192.168.18.14
TFTP_PREFIX=1
BOOT_ORDER=0x21
SD_BOOT_MAX_RETRIES=3
NET_BOOT_MAX_RETRIES=5
TFTP_PREFIX_STR=rapi4-1_boot/
```

4. 一些配置说明

- BOOT_UART

    设置为1，表示使能GPIO 14和 15的输出，也就是我们可以连接串口打开信息。其串口参数为波特率115200，8位，无奇偶校验位，1位的停止位。

- TFTP_PREFIX

    0 - Use the serial number e.g. "9ffefdef/"

    1 - Use the string specified by TFTP_PREFIX_STR

    2 - Use the MAC address e.g. "DC-A6-32-01-36-C2/"
    
    Default: 0

    自己定义TFTP_PREFIX_STR，方便识别 pi，所以设置成 1。

- TFTP_PREFIX_STR

    当TFTP_PREFIX设置为1的时候，可以设置TFTP_PREFIX_STR的路径。这样每个 pi 通过有不同的 TFTP_PREFIX_STR 从而启动不同的 rootfs 中去。

- TFTP_IP

    设置TFTP服务器的IP地址，树莓派的IP地址是通过DHCP自动获取的。

- BOOT_ORDER

    该参数配置了不同的启动模式：
    - 0x0 - NONE (stop with error pattern)
    - 0x1 - SD CARD
    - 0x2 - NETWORK

5. 将配置写回到`pieeprom.bin`
```shell
rpi-eeprom-config --out pieeprom-new.bin --config bootconf.txt pieeprom.bin
```
6. 更新eeprom
```shell
sudo rpi-eeprom-update -d -f ./pieeprom-new.bin
```

## 四、 iscsi

nfs方式无法支持docker的overlyfs。

iscsi 搭建
https://www.cnblogs.com/pipci/p/11617680.html

1. 搭建iscsi
安装tgt
`sudo apt install -y tgt`

2. 创建 rootfs img 文件

```shell
dd if=/dev/zero of=/data/pxe/rpi4-1-root.img bs=1G count=128 status=progress
```

```shell
root@rock5b:~# cat /etc/tgt/conf.d/rpi.conf 
default-driver iscsi

<target iqn.cluster.tsic.top:rpi4.rootfs1>
        backing-store /data/pxe/rpi4-1-root.img
</target>

<target iqn.cluster.tsic.top:rpi4.rootfs2>
        backing-store /data/pxe/rpi4-2-root.img
</target>

<target iqn.cluster.tsic.top:rpi4.rootfs3>
        backing-store /data/pxe/rpi4-3-root.img
</target>
```

然后使用 iscsiadm 加载iscsi 磁盘。
```shell
iscsiadm -m discovery -t st -p 192.168.18.14
iscsiadm --mode node --targetname "iqn.cluster.tsic.top:rpi4.rootfs2" --portal 192.168.18.14 --login
```
`lsblk`即可看到一个 `sdX` 磁盘。然后为该磁盘分区，将 `/` 内容同步到磁盘中即可。

2. 修改pi内核启动参数

分区时会有相关输出，或者使用 `blkid` 查询分区的 UUID。
root=UUID=xxxxxxxxxxxxxxxxxxx

```
console=serial0,115200 console=tty1 root=UUID=c48523b6-8262-416c-a5da-9dbe6656ac70 ip=192.168.18.15::192.168.18.1:255.255.255.0:rpi4-1:eth0:off ISCSI_INITIATOR=iqn.2016-04.com.open-iscsi:ca33685ee2a ISCSI_TARGET_NAME=iqn.cluster.tsic.top:rpi4.rootfs1 ISCSI_TARGET_IP=192.168.18.14 ISCSI_TARGET_PORT=3260 rw rootfstype=ext4 fsck.repair=yes rootwait elevator=deadline cgroup_memory=1 cgroup_enable=memory cgroup_enable=cpuset
```

其他一些参数
iscsi 认证相关
ISCSI_USERNAME=
ISCSI_PASSWORD=
