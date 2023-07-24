---
title: "k3s with multus cni"
date: 2023-07-24T11:38:08+08:00
draft: false
tags: [ "cni", "k8s"]
categories: ['k8s-at-home']
---

# k3s with multus cni

最近在家部署了一个k3s，用于管理一些家用服务比如jellyfin，homeassistant等，在使用homeassistant时，遇到一些网络问题，在了解k8s一点网络架构后，需要在k8s中安装multus cni，为pod创建额外网口来解决我的问题。
## 一、安装multus
拉取官方git仓库`git clone https://github.com/k8snetworkplumbingwg/multus-cni.git`

然后打开`code multus-cni`

修改`deployments/multus-daemonset.yml`

需要修改有三处
1. cniVersion

k3s 默认采用的 flannel 的 cniVersion 已经升级到 1.0.0 版本，为了 multus 可以正确生成 flannel 相关配置，需要把cniVersion提高到 1.0.0

2. kubeconfig

multus 默认读取 kubeconfig 的路径为 `/etc/cni/net.d/multus.d/multus.kubeconfig`，但是 k3s 将相关配置保存在 `/var/lib/rancher/k3s/agent/etc/` 中，导致 cni 相关的配置（下面volume）会改动到此，所以需要修改 kubeconfig 路径到 `/var/lib/rancher/k3s/agent/etc/cni/net.d/multus.d/multus`.kubeconfig

3. volume

为了配置 k3s 资源的默认保存路径
将 `/etc/cni/net.d` 修改为 `/var/lib/rancher/k3s/agent/etc/cni/net.d`
将 `/opt/multus/bin` 修改为 `/var/lib/rancher/k3s/data/current/bin`

生成的diff文件
```diff
diff --git a/deployments/multus-daemonset.yml b/deployments/multus-daemonset.yml
index ab626a66..28eb48af 100644
--- a/deployments/multus-daemonset.yml
+++ b/deployments/multus-daemonset.yml
@@ -125,7 +125,7 @@ data:
       },
       "delegates": [
         {
-          "cniVersion": "0.3.1",
+          "cniVersion": "1.0.0",
           "name": "default-cni-network",
           "plugins": [
             {
@@ -145,7 +145,7 @@ data:
           ]
         }
       ],
-      "kubeconfig": "/etc/cni/net.d/multus.d/multus.kubeconfig"
+      "kubeconfig": "/var/lib/rancher/k3s/agent/etc/cni/net.d/multus.d/multus.kubeconfig"
     }
 ---
 apiVersion: apps/v1
@@ -183,8 +183,8 @@ spec:
         command: ["/thin_entrypoint"]
         args:
         - "--multus-conf-file=auto"
-        - "--multus-autoconfig-dir=/host/etc/cni/net.d"
-        - "--cni-conf-dir=/host/etc/cni/net.d"
+        - "--cni-version=1.0.0"
+        - "--multus-kubeconfig-file-host=/var/lib/rancher/k3s/agent/etc/cni/net.d/multus.d/multus.kubeconfig"
         resources:
           requests:
             cpu: "100m"
@@ -222,10 +222,10 @@ spec:
       volumes:
         - name: cni
           hostPath:
-            path: /etc/cni/net.d
+            path: /var/lib/rancher/k3s/agent/etc/cni/net.d
         - name: cnibin
           hostPath:
-            path: /opt/cni/bin
+            path: /var/lib/rancher/k3s/data/current/bin
         - name: multus-cfg
           configMap:
             name: multus-cni-config
```
修改完后，部署 multus cni 的 daemonset 即可。
## 二、配置 cni 网络

根据我家的网络情况，我需要在homeassistant中访问lan和iot两个网段。所以定义了两个NetworkAttachmentDefinition。
一个 macvlan 对 lan 的访问，一个 vlan 对 iot 的访问。
cni中更多网络介绍 https://www.cni.dev/plugins/current/main/。
```yaml
kind: NetworkAttachmentDefinition
metadata:
  name:  lan
  namespace: homeassistant
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "eth0",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.18.0/24",
        "rangeStart": "192.168.18.32",
        "rangeEnd": "192.168.18.100",
        "routes": [
          { "dst": "192.168.18.0/24" }
        ]
      }
    }
---
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name:  iot
  namespace: homeassistant
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "vlan",
      "master": "eth0",
      "vlanId": 2,
      "linkInContainer": false,
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.32.0/24",
        "rangeStart": "192.168.32.3",
        "rangeEnd": "192.168.32.100",
        "routes": [
          { "dst": "192.168.32.0/24" }
        ]
      }
    }

```

## 三、使用 multus 
使用前需要注意，k3s 中不包含一些 cni 比如vlan，使用时会报vlan找不到的错误。
此时需要自己拉取 cni plugins 的仓库，编译对应的 cni。
`git clone https://github.com/containernetworking/plugins`
并将编译好的二进制放到 k3s data 下的 bin 里面。
然后在 pod 中加上下面内容即可。
```yaml
metadata:
    annotations:
        k8s.v1.cni.cncf.io/networks: |
        [
            {
                "name": "lan",
                "ips": ["192.168.18.32/24"],
                "namespace": "homeassistant",
                "interface": "net1",
                "mac": "32:44:0b:5a:0e:2e"
            },
            {
                "name": "iot",
                "ips": ["192.168.32.3/24"],
                "namespace": "homeassistant",
                "interface": "net1.2",
                "mac": "00:e0:4c:68:02:3d"
            }
        ]

```
这样homeassistant 就会有 loopback，eth0，net1，net1.2 四个网口。

## 四、一些问题的处理方式

1. 遇到不断加 ip 到 net1 和 eth0。从 rangeStart 直到 rangeEnd 为止。

将 multus daemonset停下，清理各个节点 multus 相关的配置，重新部署 multus。

2. 报 loopback 或者 macvlan 等非 链式调用。

在对应节点上运行 `k3s check-config`。查看 `cni` 相关的(md5sum, link)是否正常。
