[TOC]

### k8s的存储

----

#### 为什么要使用存储

- 容器的生命周期是短暂的，当容器被移除的时候，数据就会丢失
- 同一个pod下容器间需要共享文件
- 不在同一个节点的pod需要数据共享

#### 什么是volume

- k8s的容器通过volume来挂载磁盘， 存储运行时的数据，当容器被重新创建，volume挂载卷也没什么变化。
- volume和pod共同存在，当pod被删除的时候，volume也会被移除，pod可以挂载多个卷。

#### 卷的类型

- 本地卷： emptyDir， hostpath ... 。挂载在宿主机的文件，需要绑定主机，由于pod会漂移的原因，一般只用作临时存储。
- 网络数据卷： Nfs，ClusterFs，Ceph...。外部的存储卷，可以用来做持久化存储。
- 云盘： awsElastic BlockStore，azureDisk。用于挂载云服务商提供的特定存储类型。
- 内部资源： configMap、 secret、 downwardAPI。用于将K8s部分资源和集群信息公开给pod的特殊类型的卷。

#### 卷的使用

- emptyDir

  作为k8s的空目录存储卷，一般用于跨容器间的临时文件存储

```yaml
# 
apiVersion: v1
kind: Pod
metadata:
  name: emptydir
  labels:
    name: emptydir
spec:
  containers:
    - name: html-generator
      image: luksa/fortune
      volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
          readOnly: true
    - name: web
      image: nginx:alpine
      volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
          readOnly: true
  volumes:
    - name: html
      emptyDir: {}

```

由于emptydir存在于节点的特性，emptydir支持更改存储的介质，例如换成内存来存储

```yaml
  volumes:
    - name: html
      emptyDir: 
        medium: Memory
```

- hostpath

主机上预先存在的文件或目录，它们将被直接暴露给容器。 这种卷通常用于系统代理或允许查看主机的其他特权操作。大多数容器**不需要**这种卷。

```
apiVersion: v1
kind: Pod
metadata:
  name: mongodb 
spec:
  volumes:
  - name: mongodb-data
    hostPath:
      path: /tmp/mongodb
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
```

- nfs

  如果集群 是运行在自有的一组服务器上，那么就有大量其他可移植的选项用千 在卷内挂载外部存储。

```
apiVersion: v1
kind: Pod
metadata:
  name: mongodb-nfs
spec:
  volumes:
  - name: mongodb-data
    nfs:
      server: 1.2.3.4
      path: /some/path
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
```

#### PV和PVC

**持久卷（PersistentVolume，PV）** 是集群中的一块存储，可以由管理员事先制备， 或者使用[存储类（Storage Class）](https://kubernetes.io/zh-cn/docs/concepts/storage/storage-classes/)来动态制备
**持久卷申领（PersistentVolumeClaim，PVC）** 表达的是用户对存储的请求。概念上与 Pod 类似。 Pod 会耗用节点资源，而 PVC 申领会耗用 PV 资源。

![截屏2022-09-12 16.15.08](/Users/sumyf/Desktop/截屏2022-09-12 16.15.08.png)

##### 创建持久卷

```yaml
#mongodb-pv-hostpath.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv
spec:
  capacity: 
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
    - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/mongodb
```

```
kubectl get pv
```

创建持久卷时， 管理员需要告诉 Kubemetes 其对应的容量需求， 以及它是否 可以由单个节点或多个节点同时读取或写入。

![截屏2022-09-12 16.20.19](/Users/sumyf/Documents/code/docker/doc/image/截屏2022-09-12 16.20.19.png)

##### 创建持久卷声明

```yaml
# mongodb-pvc.yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc 
spec:
  resources:
    requests:
      storage: 1Gi
  accessModes:
  - ReadWriteOnce
  storageClassName: ""
```

```bash
kubectl get pvc mongodb-pvc
kubectl get pv mongodb-pv
```

PVC 状态 显示己与持久卷的 mongodb pv 绑定。请留意访 问模式 的简 写 : 

- RWO ReadWriteOnce一-仅允许单个节点挂载读写。
- ROX一-ReadOnlyMan
- y 允许多个节点挂载只读 。
- RWX一-ReadWriteMany 允许多个节点挂载读写这个卷 。

##### 在pod中使用持久卷声明

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb 
spec:
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
  volumes:
  - name: mongodb-data
    persistentVolumeClaim:
      claimName: mongodb-pvc  #持久卷声明
```

```
kubectl exec -it mongodb -- mongosh
```

##### 回收持久卷

回收策略（persistentVolumeReclaimPolicy）:

- Retain:不清理保留数据。即删除pvc或者pv后，在插件上的数据（nfs服务端）不会被删除。这种方式是最常用的，可以避免误删pvc或者pv而造成数据的丢失。
- Recycle：不保留数据。经测试pvc删除后，在nfs服务端的数据也会随机删除。**只有hostPath和NFS支持这种方式**
- Delete：删除存储资源，AWS EBS, GCE PD, Azure Disk, and Cinder volumes支持这种方式。

使用retain策略，删除pvc后，重新创建pvc，并不能使新的pvc绑定原先的pv。需要手动处理，才会恢复。

```
# 删除pod
kubectl delete pod mongodb

# 删除pvc
kubectl delete pvc mongodb-pvc

kubectl get pv 
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                STORAGECLASS   REASON   AGE
mongodb-pv   1Gi        RWO,ROX        Retain           Released   default/mongodb-pvc                               												 26m

# 即使重新创建pvc，也不会重新绑定
kubectl create -f mongodb-pvc

# 编辑pv，删除claimRef属性即可将pv变成avaliable
kubectl edit pv mongodb-pv 
```

##### 动态卷配置

通过配置storageClass来避免人工重复创建pv

```yaml
# 创建storageClass, 
#storageclass-fast-hostpath.yaml

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: k8s.io/minikube-hostpath
parameters:
  type: pd-ssd
```

```yaml
# 使用storageClass配置pvc

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc 
spec:
  storageClassName: fast
  resources:
    requests:
      storage: 100Mi
  accessModes:
    - ReadWriteOnce
```

##### 不指定存储类的动态配置

- 使用默认存储类

  ```yaml
  # 查看标准类信息， 可以看到注释栏有storageclass.kubernetes.io/is-default-class
  kubectl get sc standard -o yaml		
  
  # 创建不设置存储类的pvc
  # mongodb-pvc-dp-nostorageclass.yaml
  
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: mongodb-pvc2 
  spec:
    resources:
      requests:
        storage: 100Mi
    accessModes:
      - ReadWriteOnce
      
  # 创建后可以查看pvc使用了standard的sc
  ```


- 强制将持久卷声明绑定到预配置的其中一个持久卷

  ```yaml
  kind: PersistentVolumeClaim 
  spec:
  	storageClassName: ""   # 将空字符串指定为存储类名可确保 PVC绑定到预先配置的PV, 而不是动态配置新的 PV
  ```

  

