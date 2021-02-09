---
title: kubernetes节点（Node）状态
date: 2017-11-28 11:18:35
tags:
  - node
categories:
  - kubernetes
---

由于项目上要用节点亲和需要返回所有可用的节点。遂整理一下**Node**的属性，特别是**status**。

**Node**是**kubernetes**集群中的一台工作中的虚拟机或者物理机。每个**Node**上运行着**Docker**,**kubelet**,**kube-proxy**。
详细介绍参考[The Kubernetes Node](https://git.k8s.io/community/contributors/design-proposals/architecture/architecture.md#the-kubernetes-node)

# Node Status

## Address

- **HostName**：主机名。可以通过**kubelet**的**--hostname-override**参数重写。
- **ExternalIP**：集群外部访问**IP**。
- **InternalIP**：集群内部访问**IP**。

## Condition

**Condition**用以描述机器的运行状态。
- **OutOfDisk**：当节点上没有充足的空间创建**Pod**时值为**True**。
- **Ready**：当节点是健康的状态且可以创建**Pod**时为**True**；节点不健康且无法创建**Pod**时为**False**；而当**node controller**40s内无法检查到**Node**状态时，值为**Unknown**。

- **MemoryPressure**：当节点的内存使用存在压力，即可用内存低时值为**True**。
- **DiskPressure**：当节点的硬盘空间存在压力，即可用存储低时值为**True**。
- **NetworkUnavailable**：节点的网络配置异常时为**True**。

当**Ready**为**Unknown**或者**False**状态超过**kube-controller-manager**的参数**pod-eviction-timeout**所设定的时间（默认5分钟）时，该节点上的所有**Pod**将会被**Node Controller**删除。

## Capacity

**Capacity**用来描述节点的**CPU**，**memory**，以及可以创建**Pod**的最大数量。

##Info

通过**Kubelet**获取的节点一般信息。


# Node管理

标记一个节点使其不可被调度可以使新**Pod**不会被调度到这个节点上，同时不会影响已经在节点上运行的**Pod**。这是节点重新启动之前必要的的准备步骤。命令如下：

```
kubectl cordon $NODENAME
```

通过下面的占位**Pod**可以实现预留一部分节点资源用于处理非**Pod**进程

```
apiVersion: v1
kind: Pod
metadata:
  name: resource-reserver
spec:
  containers:
  - name: sleep-forever
    image: gcr.io/google_containers/pause:0.8.0
    resources:
      requests:
        cpu: 100m
        memory: 100Mi
```