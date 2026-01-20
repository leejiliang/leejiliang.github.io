---
layout: post
title: K8s存储三剑客的爱恨情仇：PV、PVC与StorageClass的江湖风云
categories: [k8s]
description: 本文深入剖析Kubernetes中PV、PVC和StorageClass三大核心存储概念的关系与工作原理，通过生动的比喻和实际代码示例，揭示它们如何协同解决容器持久化存储的难题，并指导在不同场景下的最佳实践。
keywords: Kubernetes, 持久化存储, PV, PVC, StorageClass
comments: false
---

# K8s存储三剑客的爱恨情仇：PV、PVC与StorageClass的江湖风云

在Kubernetes的江湖里，应用容器们来去匆匆，潇洒自如。但它们心中总有一个共同的痛：数据该如何安放？容器一重启，辛苦积累的数据便烟消云散。为了解决这个“失忆症”难题，K8s存储体系中的三位大侠应运而生：**PV（PersistentVolume，持久卷）、PVC（PersistentVolumeClaim，持久卷声明）和StorageClass（存储类）**。它们之间上演了一出精彩纷呈的“爱恨情仇”，共同守护着容器世界的数据江山。

## 一、 初识三剑客：角色与定位

在深入他们的关系之前，我们先来认识一下这三位主角。

*   **PV（PersistentVolume）**： **“资源的拥有者”**。它是集群中的一块**实实在在的存储资源**，就像云盘、NAS或者一块本地SSD。由集群管理员预先创建和配置，定义了存储的大小、访问模式（如ReadWriteOnce、ReadOnlyMany等）和具体的后端存储类型（如NFS、Ceph RBD、AWS EBS等）。PV是集群级别的资源，独立于任何Pod而存在。

*   **PVC（PersistentVolumeClaim）**： **“资源的使用者”**。它是用户（通常是应用开发者）发起的**存储资源申请**。PVC不关心存储的具体实现细节，它只声明自己需要多大的存储空间、需要什么样的访问模式。你可以把它看作是一张“存储需求清单”。

*   **StorageClass**： **“资源的自动调配者”**。它是PV的“动态创建模板”和“分类器”。当用户通过PVC申请存储时，如果指定了`StorageClass`，K8s就会根据`StorageClass`的描述，**动态地（on-demand）创建出一个符合要求的PV**，无需管理员手动干预。它定义了“如何创建PV”以及PV的通用属性。

简单比喻：如果把存储比作**房子**。
*   **PV** 就是一栋栋已经建好、装修完毕的实体房子（有100平三居室，有200平大平层）。
*   **PVC** 就是用户的租房申请单（“我需要一个至少80平，能让我一个人读写的房子”）。
*   **StorageClass** 就是一家**按需建房的房地产公司**。当你的申请单（PVC）指定了这家公司，它就会立刻按照你的要求，在后台自动盖好一栋新房子（PV）租给你。

## 二、 静态供给：PV与PVC的“传统联姻”

在StorageClass出现之前，或者在某些需要精细控制存储的场景下，PV和PVC通过“静态供给（Static Provisioning）”模式协作。

**工作流程如下：**
1.  **管理员** 预先创建一批PV，将物理存储资源“池化”。
2.  **用户** 创建PVC，提交存储需求。
3.  **K8s控制器** 扮演“月老”，根据PVC的需求（大小、访问模式），在现有的PV池中寻找一个最合适的PV。
4.  **绑定（Binding）**： 找到后，将PVC和PV进行一对一绑定。绑定后，这个PV就专属于这个PVC，不能再被其他PVC使用。
5.  **用户** 在Pod中引用这个PVC，Pod启动时，绑定的PV就会被挂载到容器内的指定路径。

**代码示例：静态供给**

首先，管理员创建一个NFS后端的PV：
```yaml
# static-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: static-nfs-pv
spec:
  capacity:
    storage: 10Gi # 存储容量
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany # 访问模式：多节点读写
  persistentVolumeReclaimPolicy: Retain # 回收策略：保留
  storageClassName: manual # 可以指定一个类名，用于筛选
  nfs:
    server: 10.244.1.100
    path: "/data/share"
```

然后，用户创建PVC来申请这个资源：
```yaml
# static-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: static-app-pvc
spec:
  storageClassName: manual # 通过类名匹配PV（可选，也可通过选择器匹配）
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi # 申请10G，必须小于等于PV的容量
```

最后，在Pod中使用这个PVC：
```yaml
# pod-with-static-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-static-storage
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: "/usr/share/nginx/html"
      name: app-storage
  volumes:
  - name: app-storage
    persistentVolumeClaim:
      claimName: static-app-pvc # 关键：引用PVC名称
```

**“爱恨”分析：**
*   **爱（优势）**： 控制力强，管理员可以精确管理每一块存储。
*   **仇（痛点）**：
    *   **运维负担重**： 管理员必须提前预估用量，创建足量的PV，容易造成资源浪费或不足。
    *   **灵活性差**： 无法实现真正的“按需分配”，与云原生理念相悖。

## 三、 动态供给：StorageClass的“降维打击”

为了解决静态供给的痛点，StorageClass携“动态供给（Dynamic Provisioning）”模式闪亮登场，彻底改变了游戏规则。

**工作流程如下：**
1.  **管理员** 预先创建好一个或多个`StorageClass`资源，里面定义了使用哪个**存储插件（Provisioner）**（如 `kubernetes.io/aws-ebs`, `csi.tencentcloud.com/cbs`）以及传递给该插件的参数（如磁盘类型、可用区）。
2.  **用户** 创建PVC，并在PVC中明确指定`storageClassName`为上面定义的某个类。
3.  **动态创建**： K8s发现这个PVC请求了一个动态StorageClass，就会触发对应的存储插件。**插件会直接与底层存储系统（如云厂商的API）通信，自动创建出一块符合要求的存储，并同时生成一个与之对应的PV**。
4.  **自动绑定**： 新创建的PV会自动与发起申请的PVC绑定。
5.  **用户** 在Pod中正常使用PVC。

**代码示例：动态供给（以AWS EBS为例）**

首先，管理员创建StorageClass（通常云厂商的K8s发行版会自带）：
```yaml
# storageclass-aws-gp3.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ebs-gp3
provisioner: ebs.csi.aws.com # CSI驱动，负责与AWS API交互
parameters:
  type: gp3 # 磁盘类型为GP3
  fsType: ext4
  iops: "3000"
  throughput: "125"
volumeBindingMode: WaitForFirstConsumer # 重要：延迟绑定，直到Pod调度
reclaimPolicy: Delete # PVC删除时，自动删除PV和底层EBS卷
allowVolumeExpansion: true # 允许卷扩容
```

用户创建PVC，指向这个StorageClass：
```yaml
# dynamic-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-app-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ebs-gp3 # 关键：指定动态StorageClass
  resources:
    requests:
      storage: 20Gi # 需要20G的GP3卷
```
**用户无需创建PV！** K8s系统会自动完成PV的创建和绑定。

**“降维打击”的优势：**
*   **自动化**： 彻底解放管理员，实现存储的按需自助服务。
*   **资源高效**： 避免闲置PV，节省成本。
*   **标准化与多样化**： 可以定义多个StorageClass（如`fast-ssd`, `slow-hdd`, `encrypted`），满足不同应用对性能、成本、安全性的差异化需求。

## 四、 爱恨交织：核心关系与最佳实践

1.  **绑定关系**： PVC与PV是**1对1的独占绑定**。一个PVC一旦绑定某个PV，直到释放前，两者都专属于对方。StorageClass不参与绑定，它只负责创建PV。

2.  **回收策略（Reclaim Policy）**： 决定PVC删除后，其绑定的PV及底层存储的命运。
    *   `Retain`（保留）： 保留PV和底层数据，手动清理。常用于静态供给或关键数据。
    *   `Delete`（删除）： 自动删除PV对象**以及后端存储设施里的数据**。**动态供给的默认策略，使用时需谨慎！**
    *   `Recycle`（回收，已废弃）： 基本被`Dynamic Provisioning`取代。

3.  **延迟绑定（`volumeBindingMode: WaitForFirstConsumer`）**： 这是StorageClass的一个关键特性。对于某些与节点相关的存储（如本地盘、特定可用区的云盘），它让PVC在**被Pod使用时**才绑定和创建PV，从而确保PV创建在Pod被调度的节点（或可用区），避免调度失败。

4.  **选择器（Selector）**： PVC可以通过`selector`字段，根据PV的标签进行更精细的匹配，这在静态供给中很有用。

**最佳实践场景：**
*   **使用动态供给作为默认选择**： 在云环境或拥有成熟存储基础设施时，优先使用StorageClass，提升运维效率。
*   **静态供给用于特殊场景**： 对接老旧存储系统、使用特定硬件、或有严格合规/安全要求需要手动审批时。
*   **合理规划StorageClass**： 根据业务需求（性能、加密、备份）创建不同的StorageClass，并通过名称或注解让用户清晰选择。
*   **务必注意回收策略**： 生产环境中，对于动态供给的重要数据存储，考虑将默认的`Delete`策略改为`Retain`，或确保有完善的备份机制。

## 五、 总结：合则两利，分则俱伤

K8s存储三剑客的故事，是一个从**手动管理**到**自动编排**的进化史。

*   **PV** 是基石，代表了存储的物理实体。
*   **PVC** 是桥梁，将用户抽象的需求与具体的存储连接起来。
*   **StorageClass** 是引擎，驱动着存储供给的自动化进程。

没有PVC，PV就无法被Pod方便地声明和使用；没有StorageClass，PV的创建就需要大量的人工操作，无法适应弹性敏捷的云原生应用。它们三者相互依存，共同构成了Kubernetes强大、灵活且可扩展的持久化存储体系。

理解它们的“爱恨情仇”，就是理解K8s如何将复杂的存储基础设施，化身为应用可简单消费的“存储即服务”。下次当你编写PVC的yaml文件时，不妨想想背后是三位大侠在为你保驾护航。