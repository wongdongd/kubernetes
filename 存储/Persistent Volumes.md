# Persistent Volumes

该文档描述了k8s中PersistentVolumes的现状。建议先熟悉volumes。

## 介绍

存储管理与实例管理相比是一个明显的问题。**PersistentVolume** 子系统为用户和管理者提供了一个API，该API从使用方法中抽象出了k8s提供存储的细节。我们将介绍两个新的API资源：**PersistentVolume** 和 **PersistentVolumeClaim**。

PV是管理员在集群中配置或者是使用Storage Class动态配置的一部分存储。它是集群中的一种资源，就像是node是一种集群资源一样。PVs是类似Volumes的卷插件，但是有独立于使用该PV的Pod的生命周期。该API对象捕获NFS，iSCSI或特定于云提供商的存储系统的存储实现的详细信息。

PVC是用户对于存储的请求。她类似于Pod。Pod消耗node资源，PVCs消耗PV资源。Pods可以请求特定级别的资源(CPU和内存)，PVC可以请求特定的大小和访问模式(例如，可以按照一次 read/write或者 多次 read-only方式挂载)。

虽然PersistentVolumeClaims允许用户使用抽象存储资源，但是对于不同的问题，用户通常需要具有不同属性（例如性能）的PersistentVolumes。集群管理员需要能够提供各种持久卷，而不仅仅是大小和访问模式，这些持久卷在更多方面有所不同，而又不会使用户了解如何实现这些卷的细节。对于这些需求，可以参考StorageClass资源。

## volume和claim的生命周期

PVs是集群资源，PVC是对于这些资源的请求，并且对这些资源进行检查。PV和PVC的交互遵循以下生命周期：

### Provisioning(配置)

两种PV的配置方式：静态、动态

* 静态
集群管理员创建PVs，携带着实际存储的详细信息，可以被用户使用。存在于k8s的API中，可以被调用。

* 动态
当管理员创建的PVs都无法满足用户的PVC时，集群会尝试动态的为PVC创建特定的volume。这个是基于StorageClassess的：PVC必须请求一个Storage Class并且管理员必须创建和配置过该storage class。Claims请求 ```""```的class会有效地阻止动态配置。

为了使基于storage class的动态配置生效，管理员需要使API服务器上的DefaultStorageClass生效.可以在 API服务组件标志```--enable-admission-plugins```的有序列表中以","分隔来添加DefaultStorageClass使其生效.

### Binding(绑定)
用户创建一个PersistentVolumeClaim，或者在动态预配置的情况下，已经创建了一个PersistentVolumeClaim，该请求具有特定的存储量和访问模式. 主服务中的控制回路监视新的PVC，找到匹配的PV（如果可能），并将它们绑定在一起.如果PV是为一个新的PVC动态配置的,那么该PV总是和该PVC绑定.否则，用户将始终获得至少他们请求的东西，但是volume可能超出了要求的数量.绑定后，无论绑定如何绑定，PersistentVolumeClaim绑定都是互斥的. PVC到PV的绑定使用ClaimRef来做一对一映射,ClaimRef是PV和PVC的双向绑定.

如果匹配的PV不存在,PVC将永远保持unbound状态.例如,一个配置了很多个50G的PV,是不会匹配一个请求100G的PVC的.

### Using(使用)
Pods使用claims作为volumes.集群检查相应的claim来查找绑定的volume,并把volume挂载到POd上.对于支持多种访问模式的volumes来说,用户在使用claim时指定所需要的访问模式.

一旦用户声明了PVC并且PVC绑定了,这个PVC就属于该用户.用户在调度Pod时可以在ods的volumes字段下添加persistentVolumeClaim来访问对应的volume.

### Storage Object in Use Protection(使用中的存储对象保护)

SOUP机制的目的是确保Pod正在使用的PVC和绑定的PVs不会被删除,因为这会导致数据丢失.

如果用户删除了Pod正在使用的PVC,该PVC不会立刻被移除.PVC的清除被推迟，直到任何Pod不再主动使用PVC.同样的,当管理员删除绑定到PVC的PV时,PV也不会立即移除,而是等到PV不再绑定到任何PVC上.

当PVC的status为**Terminating**并且**Finalizers**列表包含```kubernetes.io/pvc-protection```时,PVC时受保护的:
```
kubectl describe pvc hostpath
Name:          hostpath
Namespace:     default
StorageClass:  example-hostpath
Status:        Terminating
Volume:
Labels:        <none>
Annotations:   volume.beta.kubernetes.io/storage-class=example-hostpath
               volume.beta.kubernetes.io/storage-provisioner=example.com/hostpath
Finalizers:    [kubernetes.io/pvc-protection]
...
```
同理,PV的情况:
```
kubectl describe pv task-pv-volume
Name:            task-pv-volume
Labels:          type=local
Annotations:     <none>
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    standard
Status:          Terminating
Claim:
Reclaim Policy:  Delete
Access Modes:    RWO
Capacity:        1Gi
Message:
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /tmp/data
    HostPathType:
Events:            <none>
```

### Reclaiming(回收)

当用户完成其卷处理后，他们可以从允许回收资源的API中删除PVC对象.PersistentVolume的回收策略告诉集群在释放其声明后如何处理该卷.目前,卷支持Retained,Recycled和Deleted.

#### Retain(保留)

Retain回收策略允许资源的手动回收.当PVC被删除时,PV仍然保留并且该卷标记为"released".但是该卷不能被其他声明使用,因为其具有上一个pvc的数据.管理员可以按照以下步骤手动回收相应卷:
1. 删除PV.删除PV之后，外部基础架构中的关联存储资产（例如AWS EBS，GCE PD，Azure Disk或Cinder卷）仍然存在.
2. 手动清除相应的存储数据
3. 手动删除关联的存储资产,或者如果你想重新使用同一个存储资产,就重新使用存储资产定义创建一个新的PV.

#### Delete(删除)

对于支持Delete Reclaim策略的卷插件，删除操作会同时从Kubernetes中删除PersistentVolume对象以及外部基础架构中的关联存储资产，例如AWS EBS，GCE PD，Azure Disk或Cinder卷.动态预配置的卷继承其StorageClass的回收策略，默认为Delete.管理员应根据用户的期望配置StorageClass,否则,PV必须在创建后进行编辑和patched.

#### Recycle(回收)

如果基础卷支持Recycle回收策略,那么该策略会移除卷上的数据(rm -rf /thevolume/*)并使得该卷可以用在新的声明上.

但是,管理员可以使用k8s控制管理器命令行参数配置自定义的回收Pod模板.标准的回收Pod模板必循包含```volume```定义,如下所示:

```
apiVersion: v1
kind: Pod
metadata:
  name: pv-recycler
  namespace: default
spec:
  restartPolicy: Never
  volumes:
  - name: vol
    hostPath:
      path: /any/path/it/will/be/replaced
  containers:
  - name: pv-recycler
    image: "k8s.gcr.io/busybox"
    command: ["/bin/sh", "-c", "test -e /scrub && rm -rf /scrub/..?* /scrub/.[!.]* /scrub/*  && test -z \"$(ls -A /scrub)\" || exit 1"]
    volumeMounts:
    - name: vol
      mountPath: /scrub
```
然而,在Pod模板中```volume```部分定义的指定路径会被正在回收的volume的指定路径替代.

### Expanding Persistent Volumes Claims(扩展PVC)

默认是支持PVC的扩容的,可以对多种类型的volumes进行扩容: 
1. gcePersistentDisk
2. awsElasticBlockStore
3. Cinder
4. glusterfs
5. rbd
6. Azure File
7. Azure Disk
8. Portworx
9. FlexVolumes
10. CSI

只有在storage class的```allowVolumeExpansion```字段为true时才可以扩容PVC.
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gluster-vol-default
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://192.168.10.100:8080"
  restuser: ""
  secretNamespace: ""
  secretName: ""
allowVolumeExpansion: true
```
为了请求更大的卷,可以编辑PVC对象,设定更大的大小.这将触发支持基础PV的卷的扩展.并不会生成新的PV,相反会调整已经存储的卷的大小.

#### CSI Volume expansion(CSI卷扩展)
CSI卷扩展是默认支持的,但也需指定CSI驱动来支持卷扩展.

#### Resizing a volume containing a file system(调整包含文件系统的卷大小)
只有volume包含的文件系统是XF3, Ext3或者Ext4时才能扩缩容. 

当卷包含文件系统时,只有Pod利用ReadWrite模式使用PVC时才能对文件系统扩缩容.当Pod启动或者Pod运行且基础文件系统支持在线扩展时,文件系统扩展才能完成.

如果将```RequireFSResize```设为true,则FlexVolumes允许调整大小.可以在Pod重新启动时调整FlexVolumes的大小.

#### Resizing an in-use PersistentVolumeClaim(扩容正在使用的PVC)

> k8s在beta 1.15和alpha 1.11之后支持, ExpandInUsePersistentVolumes字段必须打开

这种情况下,不必删除和重新创建Pod或者Deployment.只要PVC的文件系统扩展完了,PVC就自动变得可用.该特征对于不被Pod或者Deployment使用的PVC没有任何影响.必须先创建一个使用PVC的Pod,才能完成扩展.

和其他volumes类似, FlexVolume卷也可以在被Pod使用时扩展.

## Type of Persistent Volumes(PV种类)
PV的种类通过插件实现,k8s目前支持以下插件:(只列了常见的)
1. NFS
2. RBD (Ceph Block Device)
3. CephFS
4. HostPath:仅用于单节点测试--不以任何方式支持本地存储,并且在多节点集群中不起作用

## PersistentVolumes

