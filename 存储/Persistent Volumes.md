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

每个PV包含一个spec和状态，即卷的spec和状态。PV对象的名称必须是有效的DNS子域名。
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```

### Capacity(容量)

通常可以在PV的 ```capacity```属性指定存储的容量。
目前只可以配置或者请求存储大小，预计将来会有IOPS、吞吐量等。

### Volume Mode

k8s支持两种PV的volumeMode： Filesystem和Block。volumeMode是一个可选的API参数，默认是Filesystem。

* 一个具有volumeMode：Filesystem的卷会被挂载到Pod的中目录。如果该卷由块设备支持并且该设备为空，则Kuberneretes会在首次挂载之前在该设备上创建一个文件系统

* 可以将volumeMode设置为Block以将卷用作原始块设备。这样的卷会以一个没有文件系统的块设备挂载进Pod。模式对于为Pod提供最快的访问卷的方式很有用，因为Pod和卷之间没有任何文件系统层。然而，运行在Pod中的应用必须知道如何使用这个块设备。详情[Raw Block Volume Support](https://kubernetes.io/docs/concepts/storage/persistent-volumes/docs/concepts/storage/persistent-volumes/#raw-block-volume-support)

### Access Modes

可以通过资源提供者支持的任何方式将PV安装在主机上。如下表所示，提供者有不同的能力，并且可以将PV的访问模式设置为特定卷支持的模式。例如：NFS可以支持多个 读/写客户端，但是一个特定的NFS PV可能以只读模式提供服务。每个PV都有自己的一组访问模式，可以描述PV的能力。

访问模式：
* ReadWriteOnce: 该卷可以通过单个节点以读写方式安装
* ReadOnlyMany: 该卷可以被许多节点只读安装
* ReadWriteMany: 该卷可以被许多节点以读写方式安装

在命令行CLI端，访问模式缩写为：
* RWO: ReadWriteOnce
* ROX: ReadOnlyMany
* RWX: ReadWriteMany

**注意**：一个卷同一时间只能使用一个访问模式，即使它支持多种。

### Class

通过设定```storageClassName```属性为StorageClass的名称，可以为PV设置一个class。具有特定class的PV将只会绑定到请求该class的PVC上。一个没有storageClassName的PV不会有class并且将只会绑定到请求没有指定class的PVC上。

在之前版本，注解```volume.beta.kubernetes.io/storage-class```用来替代storageClassName属性。目前该注解还在使用，但是将来版本将会过期。

### Reclaim Policy

目前的回收策略有：
* Retain: 手动回收
* Recycle: 基本的删除(rm -rf /thevolume/*)
* Delete: 关联的卷将被删除

目前，只有NFS和HostPath支持Recycle。AWS EBS, GCE PD, Azure Disk, 和Cinder支持Delete。

### Mount Options
当PV被安装在node上时，k8s管理员可以指定额外的挂载选项。

挂载选项未经验证，因此如果其中一个无效，挂载就会失败。类似class，注解```volume.beta.kubernetes.io/mount-options```可以用来替代mountOptions属性，但是未来会过期。

### Node Affinity

> 大多数volume都不需要设置这个属性。AWS EBS, GCE PD和Azure Disk卷类型比较流行使用这个属性。

PV可以指定节点亲和力来定义约束，以限制可以访问此卷的节点。使用PV的Pod仅会安排到由节点亲缘关系选择的节点上.

### Phase

一个卷有以下几种状态：
* Available: 还没有绑定到声明的自由资源
* Bound: 卷已绑定到声明上
* Released: 声明被删除了，但是集群还没有重新声明该资源
* Failed: 卷自动回收失败

命令行将会展示绑定到PV的PVC的名称。

## PersistentVolumeClaims
每个PVC包含一个spec和状态，即卷的spec和状态。PVC对象的名称必须是有效的DNS子域名。
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

### Access Modes(访问模式)
当请求使用特定访问模式进行存储时，声明使用与卷相同的约定。

### Volume Modes
声明使用与卷相同的约定来表示将卷作为文件系统或块设备使用。

### Resources
声明与Pods一样，可以请求特定数量的资源。在这种情况下，该请求用于存储。相同的资源模型适用于数量和声明

### Selector
声明可以指定标签选择器以进一步过滤卷集。仅其标签与选择器匹配的卷可以绑定到声明。选择器可以包含两个字段:
* matchLables: 该卷必须带有此值的标签
* matchExpressions: 通过指定键，值列表以及与键和值相关的运算符得出的要求列表。有效运算符包括In，NotIn，Exists和DidNotExist

matchLabel和matchExpressions中的所有要求都进行“与”运算-必须全部满足才能匹配.

### Class
声明可以通过使用属性storageClassName指定StorageClass的名称来请求特定类。只能将所请求类的PV（具有与PVC相同的storageClassName的PV）绑定到PVC.

PVC不是必须要请求类。始终将其storageClassName设置为""的PVC解释为请求不带类的PV，因此只能将其绑定到不带类的PV（不带注释或一组等于"）。没有storageClassName的PVC不太相同，集群会对其进行不同的处理，具体取决于是否打开了DefaultStorageClass asmission插件:
* 插件打开：管理员会指定一个默认的StorageClass。所有没有storageClassName的PVC将会绑定到该默认的PV。通过将StorageClass对象中的注释storageclass.kubernetes.io/is-default-class设置为true来指定默认的StorageClass。如果管理员未指定默认值，则集群将会按照插件关闭处理。如果指定了多个默认值，admission插件会阻止所有PVC的创建。
* 插件关闭：没有默认StorageClass的概念。所有没有storageClassName的PVC只能绑定到没有类的PV。在这种情况下，不使用storageClassName的PVC与将storageClassName设置为“”的PVC一样对待。

根据安装方法的不同，可以在安装过程中通过插件管理器将默认的StorageClass部署到Kubernetes集群。

当PVC在请求StorageClass外还指定了selector选择器，将会对这些请求做与操作：只有同时满足请求类和标签的PV才被绑定到PVC。

## Claims As Volume

Pod通过是用声明来访问存储并用做卷。声明和Pod要在同一个命名空间。集群在Pod的命名空间中寻找声明，并且使用其获得支持声明的PV。之后卷会被安装在主机和Pod中：
```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```

### A note on namespaces
PersistentVolumes绑定是排他的，并且由于PersistentVolumeClaims是命名空间对象，因此只能在一个命名空间中使用“Many”模式（ROX，RWX）挂载声明.

## Raw Block Volume Support
支持原始块卷和动态配置的插件包括：RDB...

### PersistentVolume using a Raw Block Volume
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: block-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  persistentVolumeReclaimPolicy: Retain
  fc:
    targetWWNs: ["50060e801049cfd1"]
    lun: 0
    readOnly: false
```

### PersistentVolumeClaim requesting a Raw Block Volume
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 10Gi
```
### Pod specification adding Raw Block Device path in container
```
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-block-volume
spec:
  containers:
    - name: fc-container
      image: fedora:26
      command: ["/bin/sh", "-c"]
      args: [ "tail -f /dev/null" ]
      volumeDevices:
        - name: data
          devicePath: /dev/xvda
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: block-pvc
```
### Binding Block Volumes
如果用户通过使用PersistentVolumeClaim规范中的volumeMode字段指示原始块来请求原始块卷，则绑定规则与以前的版本（将其不视为该规范的一部分）的版本略有不同。

## Volume Snapshot and Restore Volume from Snapshot Support(卷快照和从快照恢复卷)
卷快照功能仅有CSI卷插件支持。

要启用从卷快照数据源还原卷的支持，请在apiserver和controller-manager上启用VolumeSnapshotDataSource功能.

### Create a PersistentVolumeClaim from a Volume Snapshot
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restore-pvc
spec:
  storageClassName: csi-hostpath-sc
  dataSource:
    name: new-snapshot-test
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## Volume Clone
仅有CSI卷插件支持卷拷贝
### Create PersistentVolumeClaim from an existing PVC
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cloned-pvc
spec:
  storageClassName: my-csi-plugin
  dataSource:
    name: existing-src-pvc-name
    kind: PersistentVolumeClaim
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## Writing Portable Configuration(编写便携式配置)

如果您要编写在广泛的集群上运行并且需要持久存储的配置模板或示例，建议您使用以下模式:
* 将PersistentVolumeClaim对象包含在配置包中（与Deployment，ConfigMap等一起)
* 不要在配置中包括PersistentVolume对象，因为实例化配置的用户可能没有创建PersistentVolumes的权限
* 在实例化模板时，为用户提供提供storage class名称的选项:
  * 如果用户提供了storage class名称，则将该值放入persistentVolumeClaim.storageClassName字段中。如果集群具有由管理员启用的StorageClasses，这将导致PVC匹配正确的StorageClass。
  * 如果用户未提供storage class名称，则将persistentVolumeClaim.storageClassName字段保留为nil。这将导致为集群中具有默认StorageClass的用户自动设置PV。许多群集环境都安装了默认的StorageClass，或者管理员可以创建自己的默认StorageClass。

* 在您的工具中，请注意一段时间后未绑定的PVC，并将其暴露给用户，因为这可能表明集群没有动态存储支持（在这种情况下，用户应创建匹配的PV）或集群没有存储系统（在这种情况下，用户无法部署需要PVC的配置）。

