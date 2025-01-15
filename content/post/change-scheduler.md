---
title: "Change Scheduler for k3s"
date: 2023-10-08T11:12:10+08:00
draft: false
tags: ["k8s","k3s"]
categories: ['k8s-at-home']
---

本文主要介绍使用load-watcher + scheduler-plugins + descheduler 配合实现k3s根据资源占用平衡调度。

## k3s 调度问题

k8s 默认调度器不会更具节点的实际负载进行调度，只会根据request中申请的资源来。所以会导致多数的部署都是往一台比较强劲的节点上，或者新加入的节点不会有任何的pod分配过来。

## metrics-server

使用 k8s 默认的metrics-server 来获取节点资源使用情况

```shell
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

## load-watcher

为 [trimaran](https://github.com/kubernetes-sigs/scheduler-plugins/blob/master/kep/61-Trimaran-real-load-aware-scheduling/README.md) 打分时提供当前node的资源使用情况数据。
上游项目地址: https://github.com/paypal/load-watcher
镜像: https://github.com/tsic404/docker-images/pkgs/container/load-watcher

为了 load-watcher 方便读取集群各个节点的资源使用信息，将kubeconfig作为configmap传递到容器中，并指定 KUBE_CONFIG 环境变量。

```yaml
--- # kube-config configmap
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-config
  labels:
    app: load-watcher
  namespace: loadwatcher
immutable: true
data:
  kube-config: |
    xxxxx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: load-watcher-deployment
  namespace: loadwatcher
  labels:
    app: load-watcher
spec:
  replicas: 1
  selector:
    matchLabels:
      app: load-watcher
  template:
    metadata:
      labels:
        app: load-watcher
    spec:
      containers:
      - name: load-watcher
        image: ghcr.io/tsic404/load-watcher:latest
        env:
         - name: KUBE_CONFIG
           value: /kube-config
        ports:
        - containerPort: 2020
        volumeMounts:
          - name: kube-config
            mountPath: /kube-config
            subPath: kube-config
      volumes:
        - name: kube-config
          configMap:
            name: kube-config
            items:
              - key: kube-config
                path: kube-config
---
apiVersion: v1
kind: Service
metadata:
  namespace: loadwatcher
  name: load-watcher
  labels:
    app: load-watcher
spec:
  type: ClusterIP
  ports:
  - name: http
    port: 2020
    targetPort: 2020
    protocol: TCP
  selector:
    app: load-watcher
```

## scheduler-plugins

采用 helm install as a second scheduler
使用 trimaran 插件来根据当前 node 的实际资源占用情况来进行调度。
values.yaml

```yaml
scheduler:
  name: trimaran
  image: registry.k8s.io/scheduler-plugins/kube-scheduler:v0.26.7
  replicaCount: 1
  leaderElect: false

controller:
  name: scheduler-plugins-controller
  image: registry.k8s.io/scheduler-plugins/controller:v0.26.7
  replicaCount: 1

plugins:
  enabled: ["TargetLoadPacking"]
  disabled: ["NodeResourcesBalancedAllocation", "NodeResourcesLeastAllocated"]

pluginConfig:
- name: TargetLoadPacking
  args:
    watcherAddress: http://load-watcher.loadwatcher.svc.cluster.local:2020
```

```shell
helm install scheduler-plugins as-a-second-scheduler/ --create-namespace --namespace scheduler-plugins -f values.yaml
```

## descheduler

k8s 中pod一旦绑定了节点就不会在进行调度，但是pod运行过程中，pod使用的资源可能会不断变化，这样分配时的平衡就会被打破。所以引入重平衡工具`descheduler`，让其找到可以可以移除的pod并驱逐他们，重新触发k8s的调度。

```shell
helm install descheduler --namespace kube-system descheduler/descheduler --set kind=Deployment
```

## depolyment

由于采用的是将 trimaran 安装成第二调度器，所以需要在 deployment 中显示指明调度器名称

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      schedulerName: trimaran
```
