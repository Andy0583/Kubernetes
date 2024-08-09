# 動態PV建立
### 0.建立測試用NameSpace 
```
# 依據需求建立NameSpace
root@k8s1:~# kubectl create ns andy
namespace/andy created

# 查看所有NameSpace資訊
root@k8s1:~# kubectl get ns
NAME              STATUS   AGE
andy              Active   17m
default           Active   22h
kube-flannel      Active   22h
kube-node-lease   Active   22h
kube-public       Active   22h
kube-system       Active   22h
ps                Active   3h38m
```

### 1.建立Storage Class
* 查看PST Array ID資訊
  
![](https://github.com/Andy0583/Dell-CSI-for-Powerstore/blob/main/image/005.png?raw=true)

* 查看PST NAS Name資訊

![](https://github.com/Andy0583/Dell-CSI-for-Powerstore/blob/main/image/006.png?raw=true)

```
root@k8s1:~# vi sc.yaml
```
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
# 修改StorageClass Name 自定義
  name: "sc-powerstore"
provisioner: "csi-powerstore.dellemc.com"
parameters:
# 修改PST Array ID
  arrayID: "PS556db8bad5cd"
  csi.storage.k8s.io/fstype: "nfs"
# 修改PST NAS Server Name
  nasName: "andy-nas"
  allowRoot: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
```
```
root@k8s1:~# kubectl create -f sc.yaml
storageclass.storage.k8s.io/sc-powerstore created

root@k8s1:~# kubectl get sc
NAME            PROVISIONER                  RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
sc-powerstore   csi-powerstore.dellemc.com   Delete          Immediate           true                   63s
```


### 2.建立PVC
```
root@k8s1:~# vi pvc.yaml
```
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
# 修改PVC Name 自定義
  name: pvc-powerstore
# 修改Name Space
  namespace: andy
spec:
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  resources:
    requests:
# 修改此PVC需求容量大小
      storage: 300Gi
# 修改此PVC所使用的SC Name
  storageClassName: sc-powerstore
```
```
root@k8s1:~# kubectl create -f pvc.yaml
persistentvolumeclaim/pvc-powerstore created

root@k8s1:~# kubectl get pvc -n andy
NAME             STATUS   VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS    AGE
pvc-powerstore   Bound    csivol-668ed320cd   300Gi      RWX            sc-powerstore   29s

root@k8s1:~# kubectl get pv
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS    REASON   AGE
csivol-668ed320cd   300Gi      RWX            Delete           Bound    andy/pvc-powerstore   sc-powerstore            47s
```

* 查看Powerstore File system資訊，是否有正確建立

![](https://github.com/Andy0583/Dell-CSI-for-Powerstore/blob/main/image/003.png?raw=true)

### 3.建立Pod
* 建立Pod yaml內容
```
root@k8s1:~# vi pod.yaml
```
```
kind: Pod
apiVersion: v1
metadata:
  name: pod_ginnet
  namespace: andy
spec:
  volumes:
  - name: andyvol
    persistentVolumeClaim:
      claimName: pvc-powerstore
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: "/andy"
      name: andyvol
```


* 依據yaml定義內容建立Pod
```
root@k8s1:~# kubectl create -f pod.yaml
```


* 查看Pod是否建立成功
```
root@k8s1:~# kubectl get pod -n andy
NAME             READY   STATUS    RESTARTS   AGE
pod_ginnet       1/1     Running   0          20s
```


* 檢查Pod是否正確掛載PowerStore NFS
```
root@k8s1:~# kubectl exec -ti pod_ginnet -n andy -- /bin/sh

# ls
andy  boot  docker-entrypoint.d   etc   lib    media  opt   root  sbin  sys  usr
bin   dev   docker-entrypoint.sh  home  lib64  mnt    proc  run   srv   tmp  var

# df -h
Filesystem                                       Size  Used Avail Use% Mounted on
overlay                                           98G  9.4G   84G  11% /
tmpfs                                             64M     0   64M   0% /dev
192.168.131.20:/csivol-668ed320cd/common_folder  302G  1.6G  300G   1% /andy
/dev/mapper/ubuntu--vg-ubuntu--lv                 98G  9.4G   84G  11% /etc/hosts
shm                                               64M     0   64M   0% /dev/shm
tmpfs                                             16G   12K   16G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                                            7.9G     0  7.9G   0% /proc/acpi
tmpfs                                            7.9G     0  7.9G   0% /proc/scsi
tmpfs                                            7.9G     0  7.9G   0% /sys/firmware

# exit
```


### 4.刪除Pod（需先刪除才可刪除PVC）
```
root@k8s1:~# kubectl delete -f pod.yaml
pod "pod" deleted

root@k8s1:~# kubectl get pod -n andy
No resources found in andy namespace.
```


### 5.刪除PVC
```
root@k8s1:~# kubectl delete -f pvc.yaml
persistentvolumeclaim "pvc-powerstore" deleted

root@k8s1:~# kubectl get pvc -n andy
No resources found in andy namespace.

root@k8s1:~# kubectl get pv
No resources found
```


* 檢查PowerStore中，該File System使否已刪除

![](https://github.com/Andy0583/Dell-CSI-for-Powerstore/blob/main/image/004.png?raw=true)

# 靜態PV
### 1.建立Storage Class
```
root@k8s1:~# vi sc.yaml
```
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: "sc-powerstore"
  namespace: "andy"
provisioner: "csi-powerstore.dellemc.com"
parameters:
  arrayID: "PS556db8bad5cd"
  csi.storage.k8s.io/fstype: "nfs"
  nasName: "andy-nas"
  allowRoot: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
```
```
root@k8s1:~# kubectl create -f sc.yaml
storageclass.storage.k8s.io/sc-powerstore created

root@k8s1:~# kubectl get sc
NAME            PROVISIONER                  RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
sc-powerstore   csi-powerstore.dellemc.com   Delete          Immediate           true                   63s
```

### 2.建立PV
* 使用既有Powerstore filesystem，掛載為PV，需先紀錄該filesystem ID。

![](https://github.com/Andy0583/Dell-CSI-for-Powerstore/blob/main/image/007.png?raw=true)
* volumeHandle格式為<volume-id/globalID/protocol><BR>
* 本範例為<664d8fd7-d4b6-a0c0-2083-22b2c6c9dd6b/PS556db8bad5cd/nfs>

```
root@k8s1:~# vi pv.yaml
```
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: andy-staticvol
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 100Gi
  csi:
    driver: csi-powerstore.dellemc.com
    volumeHandle: 664d8fd7-d4b6-a0c0-2083-22b2c6c9dd6b/PS556db8bad5cd/nfs
  persistentVolumeReclaimPolicy: Retain
  storageClassName: sc-powerstore
  volumeMode: Filesystem
```
```
root@k8s1:~# kubectl create -f pv.yaml
persistentvolume/andy-staticvol created

rroot@k8s1:~# kubectl get pv
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS    REASON   AGE
andy-staticvol   100Gi      RWX            Retain           Available           sc-powerstore            5s
```

### 3.建立PVC

* volumeName: 指定PV名稱
```
vi pvc.yaml
```
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-static
  namespace: andy
spec:
  accessModes:
   - ReadWriteMany
  volumeMode: Filesystem
  resources:
    requests:
      storage: 60Gi
  storageClassName: sc-powerstore
  volumeName: andy-staticvol
```

```
root@k8s1:~# kubectl create -f pvc.yaml
persistentvolumeclaim/pvc-static created

root@k8s1:~# kubectl get pvc -n andy
NAME         STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS    AGE
pvc-static   Bound    andy-staticvol   100Gi      RWX            sc-powerstore   7s
```

### 4.刪除PVC與PV
```
root@k8s1:~# kubectl delete pvc pvc-static -n andy
persistentvolumeclaim "pvc-static" deleted

root@k8s1:~# kubectl delete pv andy-staticvol
persistentvolume "andy-staticvol" deleted
```

* 查看Powerstore File System，"andy-static"依舊存在

![](https://github.com/Andy0583/Dell-CSI-for-Powerstore/blob/main/image/008.png?raw=true)


# CSI Snapshot 實作

* 實作內容：
  * 建立Source PVC
  * 於Source PVC寫入資料
  * 將Source PVC進行Snapshot
  * 於Source PVCS刪除資料
  * 使用Snapshot，建立新的PVC
  * 掛載至Pod使用
  * 查看Snapshot PVC資料是否存在

### 1.建立Storage Class
```
root@k8s1:~# vi snapshot-class.yaml
```

> ```
> apiVersion: snapshot.storage.k8s.io/v1
> kind: VolumeSnapshotClass
> metadata:
>   name: ps-snapshot
> driver: "csi-powerstore.dellemc.com" 
> deletionPolicy: Delete
> ```

```
root@k8s1:~# kubectl create -f snapshot-class.yaml
volumesnapshotclass.snapshot.storage.k8s.io/ps-snapshot created

root@k8s1:~# kubectl get volumesnapshotclass
NAME          DRIVER                       DELETIONPOLICY   AGE
ps-snapshot   csi-powerstore.dellemc.com   Delete           38s
```

### 2.建立Source PVC
```
vi pvc-source.yaml
```
>```
>kind: PersistentVolumeClaim
>apiVersion: v1
>metadata:
>  name: pvc-source
>  namespace: andy
>spec:
>  accessModes:
>    - ReadWriteMany
>  volumeMode: Filesystem
>  resources:
>    requests:
>      storage: 100Gi
>  storageClassName: sc-powerstore
>```

```
root@k8s1:~# kubectl create -f pvc-source.yaml
persistentvolumeclaim/pvc-source created

root@k8s1:~# kubectl get pvc -n andy
NAME         STATUS   VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS    AGE
pvc-source   Bound    csivol-a692231178   100Gi      RWX            sc-powerstore   9s
```

### 3.建立測試用Pod
```
vi pod-source.yaml
```

>```
> kind: Pod
> apiVersion: v1
> metadata:
>   name: pod-source
>   namespace: andy
> spec:
>   volumes:
>   - name: andyvol
>     persistentVolumeClaim:
>       claimName: pvc-source
>   containers:
>   - name: nginx
>     image: nginx
>     volumeMounts:
>     - mountPath: "/andy"
>       name: andyvol
>```

```
root@k8s1:~# kubectl create -f pod-source.yaml
pod/pod-source created

root@k8s1:~# kubectl get pod -n andy
NAME         READY   STATUS    RESTARTS   AGE
pod-source   1/1     Running   0          25s
```

### 4.建立測試資料

* 於Powerstore pv上建立資料夾"1200test"
```
root@k8s1:~# kubectl exec -ti pod-source -n andy -- /bin/sh
# cd /andy
# mkdir 1200test
# ls -a
.  ..  1200test
# exit
```

### 5.將Source PVC進行Snapshot
```
vi snap.yaml
```

>```
> apiVersion: snapshot.storage.k8s.io/v1 
> kind: VolumeSnapshot
> metadata:
>   name: snap1200
>   namespace: andy
> spec:
>   volumeSnapshotClassName: ps-snapshot
>   source:
>     persistentVolumeClaimName: pvc-source
>```

```
root@k8s1:~# kubectl create -f snap.yaml
volumesnapshot.snapshot.storage.k8s.io/snap1200 created

root@k8s1:~# kubectl get volumesnapshot -n andy
NAME       READYTOUSE   SOURCEPVC    SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS   SNAPSHOTCONTENT                                    CREATIONTIME   AGE
snap1200   true         pvc-source                           100Gi         ps-snapshot     snapcontent-d1c14b80-f223-483a-b77f-9b51305a90de   7s             13s
```

* 查看Powerstore該File System上Snapshot資訊
  
![](https://github.com/Andy0583/Dell-CSI-for-Powerstore/blob/main/image/009.png?raw=true)

### 6.刪除測試資料

```
* 於Powerstore pv上刪除資料夾"1200test"

root@k8s1:~# kubectl exec -ti pod-source -n andy -- /bin/sh
# cd andy
# ls -a
.  ..  1200test
# ls -a
.  ..
# exit
```

### 7.使用Snapshot，建立新的PVC
```
vi pvc-target.yaml
```

>```
> apiVersion: v1
> kind: PersistentVolumeClaim
> metadata:
>   name: pvc-target
>   namespace: andy
> spec:
>   storageClassName: sc-powerstore
>   dataSource:
>     name: snap1200
>     kind: VolumeSnapshot
>     apiGroup: snapshot.storage.k8s.io
>   accessModes:
>     - ReadWriteMany
>   resources:
>     requests:
>       storage:  100Gi
>   volumeMode: Filesystem
>```

```
root@k8s1:~# kubectl create -f pvc-target.yaml
persistentvolumeclaim/pvc-target created

root@k8s1:~# kubectl get pvc -n andy
NAME         STATUS   VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS    AGE
pvc-source   Bound    csivol-a692231178   100Gi      RWX            sc-powerstore   41m
pvc-target   Bound    csivol-fab3842a5b   100Gi      RWX            sc-powerstore   15s
```

### 8.建立驗證使用的Pod
```
vi pod-target.yaml
```

>```
> kind: Pod
> apiVersion: v1
> metadata:
>   name: pod-target
>   namespace: andy
> spec:
>   volumes:
>   - name: andyvol
>     persistentVolumeClaim:
>       claimName: pvc-target
>   containers:
>   - name: nginx
>     image: nginx
>     volumeMounts:
>     - mountPath: "/andy"
>       name: andyvol
>```

```
root@k8s1:~# kubectl create -f pod-target.yaml
pod/pod-target created

root@k8s1:~# kubectl get pod -n andy
NAME         READY   STATUS    RESTARTS   AGE
pod-source   1/1     Running   0          25m
pod-target   1/1     Running   0          8s
```

### 9.驗證snapshot volume檔案

```
root@k8s1:~# kubectl exec -ti pod-target -n andy -- /bin/sh
# cd /andy
# mkdir 1200test
# ls -a
.  ..  1200test
# exit
```

### 10.刪除作業

```
root@k8s1:~# kubectl delete -f pod-target.yaml
pod "pod-target" deleted

root@k8s1:~# kubectl delete -f pod-source.yaml
pod "pod-source" deleted

root@k8s1:~# kubectl delete -f snap.yaml
volumesnapshot.snapshot.storage.k8s.io "snap1200" deleted

root@k8s1:~# kubectl delete -f pvc-source.yaml
persistentvolumeclaim "pvc-source" deleted

root@k8s1:~# kubectl delete -f pvc-target.yaml
persistentvolumeclaim "pvc-target" deleted
```
