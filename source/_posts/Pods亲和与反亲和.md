---
title: Pod亲和与反亲和
date: 2017-11-13 17:16:24
tags: 
  - 亲和
  - 反亲和
  - Nodeselector
categories:
  - kubernetes
---
## Nodeselector
**Nodeselector**是最简明的节点绑定方式。
通过下面的命令可以给指定的**Node**添加**label**（旧版本可能没有**label**命令，可通过其他方式添加或修改）：
```
kubectl label Node <node-name> <label-key>=<label-value>
```
如：
```
kubectl label Node node1 disktype=ssd
```
**Pod**的**yaml**如下添加了**Nodeselector**域： 
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  Nodeselector:
    disktype: ssd
```
这样执行创建命令（kubectl create -f pod.yaml）后，**Pod**会调度到有指定键值对disktype: ssd的节点上运行。
## 默认label
在kubernetes v1.4版本之后，**Node**会默认添加以下几个**label**：
- kubernetes.io/hostname
- failure-domain.beta.kubernetes.io/zone
- failure-domain.beta.kubernetes.io/region
- beta.kubernetes.io/instance-type
- beta.kubernetes.io/os
- beta.kubernetes.io/arch

可以通过下面命令查看**Node**的**labels**
```
kubectl get Node --show-labels
```

## 亲和与反亲和(Affinity and anti-affinity)
**Pod**亲和与反亲和位于**Podpec**中的**affinity**域下。
**affinity/anti-affinity**极大地扩展了**Pod**的绑定方式，提供了更多样化的调度策略。主要有以下优点：

1. 语法更加丰富，支持更多的逻辑。
2. 可以进行**软指定**，当**Node**都不满足条件时，**Pod**依然会被运行。
3. 可以根据节点上**Pod**的**label**进行调度，从而实现**Pod**的亲和与反亲和。

亲和与反亲和包含**节点亲和（node affinity）**和**Pod亲和与反亲和（inter-pod affinity/anti-affinity）**两个类型。前者与**Nodeselector**类似，是针对**Node**进行绑定，但是具有亲和的前两个优点。而后者则是针对**Pod**的调度，兼具三个优点。

- Nodeselector现版本仍然有效，但最终会被亲和所取代。

### 节点亲和（Node affinity）
节点亲和出现在kubernetes v1.2α版本。其包含两种类型：**requiredDuringSchedulingIgnoredDuringExecution**和**preferredDuringSchedulingIgnoredDuringExecution**。前者指定的规则必须硬性满足（硬亲和），而后者的规则是作为偏好，调度器会尝试但不保证满足（软亲和）。名字中的"DuringSchedulingIgnoredDuringExecution"是指**Pod**运行中如果**Node**不再满足亲和规则，**Pod**仍然会在该**Node**上运行。未来版本**kubernetes**将计划提供**requiredDuringSchedulingRequiredDuringExecution**参数，用以驱逐**Node**上不再满足亲和规则的**Pod**。
看下面这个例子：

```
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        NodeselectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: gcr.io/google_containers/pause:2.0
```
该**Pod**会在满足**kubernetes.io/e2e-az-name**值为**e2e-az1**或**e2e-az2**的**Node**中优先调度到**another-node-label-key**值为**another-node-label-value**的**Node**上。
在例子中看到**operator**值为**In**，同时它还支持**NotIn**，**Exists**，**DoesNotExist**，**Gt**，**Lt**。我们可以通过**NotIn**，**DoesNotExist**来实现**Node**的反亲和。使用**node affinity**需要注意的几点如下：
- **Nodeselector**和**nodeAffinity**同时设定时，两个规则同时生效。
- 当**nodeAffinity**下有多个**NodeselectorTerms**时，他们之间是**或**的关系，只满足其中一个**NodeselectorTerms**的**Node**都符合亲和规则。
- 当**NodeselectorTerms**下有多个**matchExpressions**时，他们之间是**和**的关系，满足全部**matchExpressions**的**Node**才符合亲和规则。
- 当**Node**的**label**改变而不满足规则时，**Pod**不会被驱逐，即亲和规则只在**Pod**启动时做一次判断。

更多信息请参考[Node affinity and Nodeselector](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/nodeaffinity.md)

### Pod亲和与反亲和（Inter-pod affinity and anti-affinity）

Pod亲和与反亲和出现在kubernetes v1.4版本。
与**Node**亲和相同，**Pod**亲和与反亲和也包含**requiredDuringSchedulingIgnoredDuringExecution**和**preferredDuringSchedulingIgnoredDuringExecution**两种类型，分别对应硬亲和和软亲和。

- Pod亲和与反亲和需要大量的处理，这样会显著减慢大集群中的调度。我们不建议在大于几百个节点的集群中使用它们。
- 建议当两个**Pod**相互之间访问较多时可使用硬亲和使他们能在同一**Node**上运行。而**Pod**反亲和时因为有可能**Pod**数量会大于节点**Node**数，建议使用软亲和。

看下面这个例子：

```
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: failure-domain.beta.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: kubernetes.io/hostname
  containers:
  - name: with-pod-affinity
    image: gcr.io/google_containers/pause:2.0
```

这个**Pod**的调度规则如下：
**Pod**亲和为硬亲和，要求部署到的**Node**必须包含带有**failure-domain.beta.kubernetes.io/zone**的**key**，且该**Node**上运行着包含**security: S1**的**Pod**
**Pod**反亲和为软亲和，要求**Pod**优先部署到不满足条件（**Node**包含带有**kubernetes.io/hostname**的**key**，且该**Node**上运行着包含**security: S2**的**Pod**）的**Node**上

虽然**topologyKey**基本可以是任何合法值，但在考虑到性能和安全方面，有如下的限制：

- 反亲和为硬亲和（**RequiredDuringScheduling**）时，**topologyKey**不能为空。
- **[Admission Controllers](https://kubernetes.io/docs/admin/admission-controllers)**中设置**[LimitPodHardAntiAffinity](https://kubernetes.io/docs/admin/admission-controllers/#limitpodhardantiaffinity)**启用时，反亲和为硬亲和（**RequiredDuringScheduling**）的**topologyKey**只能为**kubernetes.io/hostname**。
- 在反亲和为软亲和（**PreferredDuringScheduling**）时，**topologyKey**为空时则指代所有的拓扑结构。但现在“所有的拓扑结构”被限制为以下几个**topologyKey**：**kubernetes.io/hostname**, **failure-domain.beta.kubernetes.io/zone**和**failure-domain.beta.kubernetes.io/region**。
- 除了以上几种情况，**topologyKey**可以是任意合法值。

## 相关文档

- [https://kubernetes.io/docs/concepts/configuration/assign-pod-node/](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/)
- [https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/nodeaffinity.md](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/nodeaffinity.md)
- [https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/podaffinity.md](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/podaffinity.md)
- [https://kubernetes.io/docs/admin/admission-controllers/#limitpodhardantiaffinity](https://kubernetes.io/docs/admin/admission-controllers/#limitpodhardantiaffinity)