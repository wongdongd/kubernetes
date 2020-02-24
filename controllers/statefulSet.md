## 概念

StatefulSet是用来管理有状态的服务的。首先明确statefulSet是controller层级的，不能够对外暴露服务，主要功能是调度和扩展各个Pod，并且保证这些Pods的顺序性和唯一性。

和Deployment类似，statefulSet管理者一组具有相同的container spec，但区别是：statefulset和每个pod都是粘性绑定的。每个Pod虽然spec一致，但是不能互换，每个Pod都有一个永久性标识，在rebuild的时候也会维护。Deployment和ReplicaSet更适合无状态服务。

## 使用场景

statefulSet通常应用在以下场景中：

* 唯一、稳定的网络标识
* 持久化、稳定的存储
* 有序，顺畅的部署和扩展
* 有序、自动的滚动更新

受限：

* Pod的存储必须由PV根据请求的storage class配置，或者由admin预先配置好
* delete或者scale操作不会将statefulSet的volumes删除，这么做是为了保证数据的安全性，比彻底删除statefulSet相关资源更有意义
* StatefulSets目前需要一个headless service来负责Pods的对外服务，这个service需要我们创建
* 当删除statefulSet时，不能保证相关的Pod会终止。为了有序、优雅的终止Pod服务，最好是scale to 0
* 当使用Pod的默认管理策略(OrderedReady)进行滚动更新时，可能会产生故障，需要手动干预修复

## 编写一个statefulSet

以下的示例演示了statefulSet的组件：
```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # by default is 1
  template:
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
```

在上例中：
* 名为nginx的headless service服务用来控制网络
* 名为web的statefulSet定义了spec：3个nginx容器副本将会运行在不同的Pod当中
* volumeClaimTemplates通过PersistentVolumn Provisioner配置的PersistentVolumn来提供稳定的存储。

### Pod Selector

在statefulSet中，.spec.selector必须与其 .spec.template.metadata.labels能够match labels。k8s 1.8版本之前，.spec.selector当忽略时按照默认的走；1.8及之后，将导致statefulSet创建时验证出错而失败。

### Pod Identity



