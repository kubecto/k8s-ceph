# k8s-ceph
参考社区对接：https://github.com/kubernetes-retired/external-storage

https://github.com/kubernetes-retired/external-storage/tree/master/ceph/rbd

其他参加链接对接rbd

参考https://medium.com/velotio-perspectives/an-innovators-guide-to-kubernetes-storage-using-ceph-a4b919f4e469

目前社区不维护了，如果使用1.28的集群，则需要使用

社区文档说明

https://kubernetes.io/docs/concepts/storage/volumes/#cephfs

- CephFS CSI 驱动程序。
- RBD CSI 驱动程序。

检查ceph状态

```groovy
ceph -s
  cluster:
    id:     20bbaa39-b5fa-4eba-bc13-9e01e3ecb0da
    health: HEALTH_WARN
            mons are allowing insecure global_id reclaim

  services:
    mon: 3 daemons, quorum ceph0,ceph1,ceph2 (age 15m)
    mgr: ceph0(active, since 30s), standbys: ceph1, ceph2
    mds: cephfs:1 {0=ceph0=up:active} 2 up:standby
    osd: 6 osds: 6 up (since 7m), 6 in (since 7m)
    rgw: 3 daemons active (ceph0.rgw0, ceph1.rgw0, ceph2.rgw0)

  task status:

  data:
    pools:   6 pools, 96 pgs
    objects: 214 objects, 6.0 KiB
    usage:   306 GiB used, 594 GiB / 900 GiB avail
    pgs:     96 active+clean
```

创建rbd存储控制器

```groovy
# cat rbd-provisioner.yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rbd-provisioner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
  - apiGroups: [""]
    resources: ["services"]
    resourceNames: ["kube-dns","coredns"]
    verbs: ["list", "get"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rbd-provisioner
subjects:
  - kind: ServiceAccount
    name: rbd-provisioner
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: rbd-provisioner
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: rbd-provisioner
  namespace: kube-system
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: rbd-provisioner
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: rbd-provisioner
subjects:
- kind: ServiceAccount
  name: rbd-provisioner
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rbd-provisioner
  namespace: kube-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rbd-provisioner
  namespace: kube-system
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: rbd-provisioner
  template:
    metadata:
      labels:
        app: rbd-provisioner
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: rbd-provisioner
        image: "quay.io/external_storage/rbd-provisioner:latest"
        env:
        - name: PROVISIONER_NAME
          value: ceph.com/rbd
      serviceAccount: rbd-provisioner
```

确保控制器启动成功

```groovy
#  kubectl get pods -l app=rbd-provisioner -n kube-system
NAME                              READY   STATUS    RESTARTS   AGE
rbd-provisioner-f5b9cd8c6-lxfbh   1/1     Running   0          14h
```

如果未启动成功

报错

```groovy
unexpected error getting claim reference: selfLink was empty, can't make reference
```

解决方案：
将RemoveSelfLink=false设置为关闭，则不会影响外置存储的使用

```groovy
spec:
  containers:
  - command:
    - kube-apiserver
添加这一行：
- --feature-gates=RemoveSelfLink=false
：重置启动
/etc/kubernetes/manifests/kube-apiserver.yaml
kubectl apply -f /etc/kubernetes/manifests/kube-apiserver.yaml
```

配置程序启动后，配置程序需要用于存储配置的管理密钥。您可以运行以下命令来获取管理密钥：

```groovy
ceph auth get-key client.admin |base64
QVFER0ZiSmx6S0p6SUJBQVpXcy82emREeE9zMFdxNjRySVpzZkE9PQ==

加密设置secret
# cat secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
  namespace: kube-system
type: kubernetes.io/rbd
data:
  key: QVFER0ZiSmx6S0p6SUJBQVpXcy82emREeE9zMFdxNjRySVpzZkE9PQ==

kubectl apply -f secret.yaml
```

如果不进行加密，会报错

```groovy
运行的pod报错
: fail to check rbd image status with: (fork/exec /usr/bin/rbd: invalid argument), rbd output: ()
```

需要加密base64

让我们为 Kubernetes 和新的客户端密钥创建一个单独的 Ceph 池：

```groovy
# ceph --cluster ceph osd pool create kube 1024 1024
Error ERANGE:  pg_num 1024 size 2 would mean 2304 total pgs, which exceeds max 1500 (mon_max_pg_per_osd 250 * num_in_osds 6)
[root@ceph0 ~]# ceph --cluster ceph osd pool create kube 256 256
pool 'kube' created
[root@ceph0 ~]# ceph --cluster ceph auth get-or-create client.kube mon 'allow r' osd 'allow rwx pool=kube'
```

获取我们在上述命令中创建的身份验证令牌，并为 kube 池的新客户端密钥创建 kubernetes 密钥。

```groovy
[root@ceph0 ~]# ceph --cluster ceph auth get-key client.kube |base64
QVFBWFJMSmxpRlBNR3hBQUdLM2EwTkZidlY3Q3liZU5LRHUzVFE9PQ==

# cat kube-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret-kube
  namespace: kube-system
type: kubernetes.io/rbd
data:
  key: QVFBWFJMSmxpRlBNR3hBQUdLM2EwTkZidlY3Q3liZU5LRHUzVFE9PQ==

	kubectl apply -f kube-secret.yaml
```

创建存储类

```groovy
# cat storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-rbd
provisioner: ceph.com/rbd
parameters:
  monitors: 10.102.28.50:6789, 10.102.28.51:6789, 10.102.28.52:6789
  adminId: admin
  adminSecretName: ceph-secret
  adminSecretNamespace: kube-system
  pool: kube
  userId: kube
  userSecretName: ceph-secret-kube
  userSecretNamespace: kube-system
  imageFormat: "2"
  imageFeatures: layering

kubectl apply -f storageclass.yaml
```

现在一切都准备好了。我们可以通过创建 PVC 来测试 Ceph-RBD。创建 PVC 后，PV 将自动创建。现在让我们创建 PVC：

```groovy
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: rbd-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: fast-rbd
```

```groovy
kubectl get pvc
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
rbd-claim   Bound    pvc-6a5a25e4-7683-4571-9681-253d50a578d3   1Gi        RWO            fast-rbd       14h
[root@k8s1 rbd]# kubectl get sc
NAME               PROVISIONER        RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
dynamic-cephfs     ceph.com/cephfs    Delete          Immediate              false                  15h
fast-rbd           ceph.com/rbd       Delete          Immediate              false                  14h
openebs-device     openebs.io/local   Delete          WaitForFirstConsumer   false                  23h
openebs-hostpath   openebs.io/local   Delete          WaitForFirstConsumer   false                  23h
# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                                     STORAGECLASS       REASON   AGE
pvc-6a5a25e4-7683-4571-9681-253d50a578d3   1Gi        RWO            Delete           Bound      default/rbd-claim                         fast-rbd                    14h
```

创建pod

```groovy
# cat nginx-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod1
  labels:
    name: nginx-pod1
spec:
  tolerations:
  - key: node-role.kubernetes.io/master
    effect: NoSchedule
  containers:
  - name: nginx-pod1
    image: nginx:1.14
    ports:
    - name: web
      containerPort: 80
    volumeMounts:
    - name: cephrbd
      mountPath: /usr/share/nginx/html
  volumes:
  - name: cephrbd
    persistentVolumeClaim:
      claimName: rbd-claim
```

这里指定前面创建的pvc的名称

查看pod运行状态

```groovy
kubectl get po
NAME                    READY   STATUS      RESTARTS   AGE
nginx-pod1              1/1     Running     0          32m
```

查看状态如何为

fail to check rbd image status with: (executable file not found in $PATH), rbd output:

```groovy
Events:
  Type     Reason                  Age                From                     Message
  ----     ------                  ----               ----                     -------
  Normal   Scheduled               99s                default-scheduler        Successfully assigned default/nginx-pod1 to master1
  Normal   SuccessfulAttachVolume  99s                attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-32da161f-026a-4196-9d49-6022a1bc8740"
  Warning  FailedMount             29s (x8 over 93s)  kubelet                  MountVolume.WaitForAttach failed for volume "pvc-32da161f-026a-4196-9d49-6022a1bc8740" : fail to check rbd image status with: (executable file not found in $PATH), rbd output: ()
```

后期测试准备rbd的二进制是否可行

则需要在机器上安装ceph-common,因为rbd 实用程序是 ceph-common 包的一部分。

参考地址：https://github.com/kubernetes-retired/external-storage/issues/1002

```groovy
yum -y install ceph-common
可以看到对应rbd的客户端工具，就像对接nfs一样，需要安装nfs-utils才能使mount正常

Installed:
  ceph-common.x86_64 1:10.2.5-4.el7

Dependency Installed:
  avahi-libs.x86_64 0:0.6.31-20.el7        boost-iostreams.x86_64 0:1.53.0-28.el7           boost-program-options.x86_64 0:1.53.0-28.el7                boost-random.x86_64 0:1.53.0-28.el7     boost-regex.x86_64 0:1.53.0-28.el7     cups-client.x86_64 1:1.6.3-52.el7_9
  cups-libs.x86_64 1:1.6.3-52.el7_9        gdisk.x86_64 0:0.8.10-3.el7                      hdparm.x86_64 0:9.43-5.el7                                  libicu.x86_64 0:50.2-4.el7_7            librados2.x86_64 1:10.2.5-4.el7        librbd1.x86_64 1:10.2.5-4.el7
  m4.x86_64 0:1.4.16-10.el7                patch.x86_64 0:2.7.1-12.el7_7                    psmisc.x86_64 0:22.20-17.el7                                python-rados.x86_64 1:10.2.5-4.el7      python-rbd.x86_64 1:10.2.5-4.el7       python-requests.noarch 0:2.6.0-10.el7
  python-urllib3.noarch 0:1.10.2-7.el7     redhat-lsb-core.x86_64 0:4.1-27.el7.centos.1     redhat-lsb-submod-security.x86_64 0:4.1-27.el7.centos.1     spax.x86_64 0:1.5.2-13.el7
```

查看pod挂载卷的位置

```groovy
df -h
/dev/rbd0                976M  2.6M  958M   1% /var/lib/kubelet/plugins/kubernetes.io/rbd/mounts/kube-image-kubernetes-dynamic-pvc-cf046a6b-bb74-11ee-b484-5620346dcd5e
```

修改文件内容

```groovy
kubectl exec -ti nginx-pod1 -- /bin/sh -c 'echo This is from CephRBD!!! > /usr/share/nginx/html/index.html'
```

访问pod

```groovy
kubectl get po -o wide
NAME                    READY   STATUS      RESTARTS   AGE     IP               NODE   NOMINATED NODE   READINESS GATES
nginx-pod1              1/1     Running     0          43m     172.90.166.250   k8s1   <none>           <none>

[root@k8s1 kube]# curl 172.90.166.250
This is from CephRBD!!!
```

删除pod，挂载目录则删除，但是pv，和pvc不会删除，数据进行了持久化

```groovy
[root@k8s1 rbd]# cd /var/lib/kubelet/plugins/kubernetes.io/rbd/mounts/kube-image-kubernetes-dynamic-pvc-cf046a6b-bb74-11ee-b484-5620346dcd5e
-bash: cd: /var/lib/kubelet/plugins/kubernetes.io/rbd/mounts/kube-image-kubernetes-dynamic-pvc-cf046a6b-bb74-11ee-b484-5620346dcd5e: No such file or directory
[root@k8s1 rbd]# ls /var/lib/kubelet/plugins/kubernetes.io/rbd
mounts
[root@k8s1 rbd]# ls /var/lib/kubelet/plugins/kubernetes.io/rbd/mounts/
[root@k8s1 rbd]# ls
kube-secret.yaml  nginx-pod.yaml  pvc.yaml  rbd-provisioner.yaml  storageclass.yaml
[root@k8s1 rbd]# ls /var/lib/kubelet/plugins/kubernetes.io/rbd/mounts/
[root@k8s1 rbd]#

[root@k8s1 rbd]# kubectl get pvc,pv
NAME                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/rbd-claim   Bound    pvc-6a5a25e4-7683-4571-9681-253d50a578d3   1Gi        RWO            fast-rbd       14h

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                                     STORAGECLASS       REASON   AGE
persistentvolume/pvc-6a5a25e4-7683-4571-9681-253d50a578d3   1Gi        RWO            Delete           Bound      default/rbd-claim                         fast-rbd                    14h
```

再次创建pod,并检查我们的数据是否存在

```groovy
[root@k8s1 rbd]# kubectl apply -f nginx-pod.yaml
pod/nginx-pod1 created
[root@k8s1 rbd]# kubectl get po -o wide
NAME                    READY   STATUS              RESTARTS   AGE     IP               NODE   NOMINATED NODE   READINESS GATES
nginx-pod1              0/1     ContainerCreating   0          4s      <none>           k8s1   <none>           <none>
sender-28437260-d954f   0/1     Completed           0          2m10s   172.90.166.246   k8s1   <none>           <none>
sender-28437261-lgth5   0/1     Completed           0          70s     172.90.166.240   k8s1   <none>           <none>
sender-28437262-p455t   0/1     Completed           0          10s     172.90.166.247   k8s1   <none>           <none>
[root@k8s1 rbd]# kubectl get po -o wide
NAME                    READY   STATUS      RESTARTS   AGE     IP               NODE   NOMINATED NODE   READINESS GATES
nginx-pod1              1/1     Running     0          6s      172.90.166.248   k8s1   <none>           <none>
sender-28437260-d954f   0/1     Completed   0          2m12s   172.90.166.246   k8s1   <none>           <none>
sender-28437261-lgth5   0/1     Completed   0          72s     172.90.166.240   k8s1   <none>           <none>
sender-28437262-p455t   0/1     Completed   0          12s     172.90.166.247   k8s1   <none>           <none>
[root@k8s1 rbd]# curl 172.90.166.248
This is from CephRBD!!!
```

查看ceph rbd存储的数据，因为前面单独创建的kube 的pool,所以数据都会存储在这

```groovy
[root@ceph0 ~]# rbd -p kube  ls
kubernetes-dynamic-pvc-60e3c0f5-bb74-11ee-b484-5620346dcd5e
kubernetes-dynamic-pvc-cf046a6b-bb74-11ee-b484-5620346dcd5e
kubernetes-dynamic-pvc-dda65ae3-bbf3-11ee-a50b-1256b9eb70a1

这个对应来自节点上df -h 
/dev/rbd0                976M  2.6M  958M   1% /var/lib/kubelet/plugins/kubernetes.io/rbd/mounts/kube-image-kubernetes-dynamic-pvc-cf046a6b-bb74-11ee-b484-5620346dcd5e
```

通过rados也可以查看对应的存储的数据

```groovy
[root@ceph0 ~]# rados -p kube ls
rbd_data.16446b8b4567.0000000000000060
rbd_data.16146b8b4567.0000000000000000
rbd_data.16146b8b4567.0000000000000086
rbd_data.16446b8b4567.00000000000000ff
rbd_data.16146b8b4567.0000000000000083
rbd_data.16446b8b4567.0000000000000087
rbd_data.16446b8b4567.0000000000000084
rbd_data.16146b8b4567.0000000000000082
rbd_data.16146b8b4567.0000000000000004
rbd_id.kubernetes-dynamic-pvc-dda65ae3-bbf3-11ee-a50b-1256b9eb70a1
rbd_data.16146b8b4567.00000000000000a0
rbd_data.16446b8b4567.0000000000000004
rbd_directory
rbd_data.16146b8b4567.0000000000000020
rbd_data.16146b8b4567.0000000000000084
rbd_header.160e6b8b4567
rbd_data.16446b8b4567.0000000000000000
rbd_data.16146b8b4567.00000000000000e0
rbd_info
rbd_data.16146b8b4567.0000000000000080
rbd_data.16446b8b4567.0000000000000081
rbd_header.16146b8b4567
rbd_data.16146b8b4567.0000000000000085
rbd_data.16146b8b4567.0000000000000021
rbd_data.16146b8b4567.0000000000000087
rbd_data.16446b8b4567.00000000000000e0
rbd_data.16146b8b4567.00000000000000ff
rbd_data.16446b8b4567.0000000000000020
rbd_id.kubernetes-dynamic-pvc-cf046a6b-bb74-11ee-b484-5620346dcd5e
rbd_data.16446b8b4567.0000000000000086
rbd_header.16446b8b4567
rbd_id.kubernetes-dynamic-pvc-60e3c0f5-bb74-11ee-b484-5620346dcd5e
rbd_data.16446b8b4567.0000000000000082
rbd_data.16446b8b4567.0000000000000080
rbd_data.16446b8b4567.0000000000000083
rbd_data.16446b8b4567.0000000000000085
rbd_data.16146b8b4567.0000000000000060
rbd_data.16146b8b4567.0000000000000081
rbd_data.16446b8b4567.00000000000000a0
```

数据存储方面

- 保留数据

可删除pod,删除pod后，不会删除pv的数据，也不会删除rbd的数据

重建pod可以使用之前的pvc绑定pv里面的数据

- 删除数据

删除pod,并删除pvc，配置reclaimPolicy: Delete 自动删除绑定的pv，但不能删除rdb的数据

```groovy
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-rbd
provisioner: ceph.com/rbd
parameters:
  monitors: 10.102.28.50:6789, 10.102.28.51:6789, 10.102.28.52:6789
  adminId: admin
  adminSecretName: ceph-secret
  adminSecretNamespace: kube-system
  pool: kube
  userId: kube
  userSecretName: ceph-secret-kube
  userSecretNamespace: kube-system
  imageFormat: "2"
  imageFeatures: layering
reclaimPolicy: Delete
```

需要手动删除ceph的数据

找出关联的pvc,使用rdb命令删除镜像数据

```groovy
rbd -p kube rm kubernetes-dynamic-pvc-14a7f7a1-bbfc-11ee-b484-5620346dcd5e
Removing image: 100% complete...done.
```

例子全部删除,使用for循环

```groovy
for i in $(rbd -p kube  ls);do rbd -p kube rm $i;done
Removing image: 100% complete...done.
Removing image: 100% complete...done.
Removing image: 100% complete...done.
Removing image: 100% complete...done.
Removing image: 100% complete...done.
Removing image: 100% complete...done.
Removing image: 100% complete...done.
Removing image: 100% complete...done.
```

pvc 数据扩展

PersistentVolume 可以配置为可扩展。这允许您通过编辑相应的 PVC 对象来调整卷的大小，请求新的更大的存储量。

当底层 StorageClass 将该字段设置为 true 时，以下类型的卷支持卷扩展`allowVolumeExpansion`。

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/e3b5ddd8-3a09-4d0b-855a-db1249d88e22/5050200f-ef0a-42c3-965a-0c56f9ab0d2b/Untitled.png)

支持扩展的话，需要在storageclass进行声明，如果提供的代码支持，那么就可以支持扩展

当前的storageclass配置

```groovy
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-rbd
provisioner: ceph.com/rbd
parameters:
  monitors: 10.102.28.50:6789, 10.102.28.51:6789, 10.102.28.52:6789
  adminId: admin
  adminSecretName: ceph-secret
  adminSecretNamespace: kube-system
  pool: kube
  userId: kube
  userSecretName: ceph-secret-kube
  userSecretNamespace: kube-system
  imageFormat: "2"
  imageFeatures: layering
reclaimPolicy: Delete
#allowVolumeExpansion: true
```

```groovy
[root@k8s1 rbd]# kubectl get pvc
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
rbd-claim   Bound    pvc-a77844e2-6780-4751-948f-100f99bdbb21   1Gi        RWO            fast-rbd       9s
[root@k8s1 rbd]# kubectl edit pvc rbd-claim
error: persistentvolumeclaims "rbd-claim" could not be patched: persistentvolumeclaims "rbd-claim" is forbidden: only dynamically provisioned pvc can be resized and the storageclass that provisions the pvc must support resize
You can run `kubectl replace -f /tmp/kubectl-edit-2847686707.yaml` to try this update again
```

当我去创建pvc之后，进行修改request的大小的时候，这里提供rbd-claim”被禁止:只有动态配置的PVC可以调整大小，并且提供PVC的存储类必须支持调整大小

修改参数增加allowVolumeExpansion: true，可以修改成功

```groovy
kubectl edit pvc rbd-claim
persistentvolumeclaim/rbd-claim edited
```

开启后，可以看到ALLOWVOLUMEEXPANSION 为true

```groovy
kubectl get sc
NAME               PROVISIONER        RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
dynamic-cephfs     ceph.com/cephfs    Delete          Immediate              false                  20h
fast-rbd           ceph.com/rbd       Delete          Immediate              true                   12m
openebs-device     openebs.io/local   Delete          WaitForFirstConsumer   false                  29h
openebs-hostpath   openebs.io/local   Delete          WaitForFirstConsumer   false                  29h
```

查看pvc大小

```groovy
kubectl get pvc rbd-claim -o yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: n
      {"api.io/bind-completed: "yes"
    pv.kubernetes.io/bound-by-controller: "yes"
    volume.beta.kubernetes.io/storage-provisioner: ceph.com/rbd
    volume.kubernetes.io/storage-provisioner: ceph.com/rbd
    volume.kubernetes.io/storage-resizer: kubernetes.io/rbd
  creationTimestamp: "2024-01-26T06:56:31Z"
  finalizers:
  - kubernetes.io/pvc-protection
  name: rbd-claim
  namespace: default
  resourceVersion: "314938"
  selfLink: /api/v1/namespaces/default/persistentvolumeclaims/rbd-claim
  uid: d2e9a5f9-4f86-4b70-8b5a-aae44ffe6e3e
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
storageClassName: fast-rbd
  volumeMode: Filesystem
  volumeName: pvc-d2e9a5f9-4f86-4b70-8b5a-aae44ffe6e3e
status:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 1Gi
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2024-01-26T06:57:02Z"
    status: "True"
    type: Resizing
  phase: Bound
```

在 Kubernetes 中进行 PVC 扩容后，**`kubectl get pvc`** 可能仍然显示之前的存储容量。这是因为 PVC 扩容是一个异步过程，而 Kubernetes 通过探测机制（probing mechanism）来检测 PVC 的状态变化。

你在 YAML 输出中看到的 **`conditions`** 字段中有一个类型为 **`Resizing`** 的条件，表示 PVC 正在进行扩容。在这个条件变为 **`True`** 之前，**`kubectl get pvc`** 可能仍然显示之前的存储容量。

查看pvc的describe 的状态

```groovy
kubectl describe pvc rbd-claim
Normal   ExternalProvisioning   7m51s (x3 over 7m51s)  persistentvolume-controller                                                        waiting for a volume to be created, either by external provisioner "ceph.com/rbd" or manually created by system administrator
  Normal   Provisioning           7m51s                  ceph.com/rbd_rbd-provisioner-f5b9cd8c6-lxfbh_33b7a230-bb73-11ee-b484-5620346dcd5e  External provisioner is provisioning volume for claim "default/rbd-claim"
  Normal   ProvisioningSucceeded  7m51s                  ceph.com/rbd_rbd-provisioner-f5b9cd8c6-lxfbh_33b7a230-bb73-11ee-b484-5620346dcd5e  Successfully provisioned volume pvc-d03e3e98-239a-4f3c-a08e-90ca4b5080c6
  Warning  VolumeResizeFailed     110s (x17 over 7m18s)  volume_expand                                                                      error expanding volume "default/rbd-claim" of plugin "kubernetes.io/rbd": rbd info failed, error: executable file not found in $PATH
```

https://github.com/kubernetes/kubernetes/issues/69082

https://github.com/kubernetes-retired/external-storage/issues/992#issuecomment-555897900

根据社区的讨论，此问题，还需要修改代码，当前他们的使用的hyperkube构建的，镜像可以放进ceph的配置，kubeadm的controller-manager为精简镜像，所以无法使用此方法，需要重新构建镜像，安装rbd的配置来操作ceph集群来扩容集群，但是待验证

cephfs 需要参考之前的代码

官方文档说明对接其他外置存储：https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner
