# Unity CSI 安裝程序
### 1.每台Node設定iSCSI連接
```
echo "InitiatorName=`/sbin/iscsi-iname`" > /etc/iscsi/initiatorname.iscsi
iscsiadm -m discovery -t st -p 172.22.46.233
iscsiadm --mode node --portal 172.22.46.233:3260,1 iqn.1992-04.com.emc:cx.virt2418y8kp91.a3 --login
systemctl restart open-iscsi
```


### 2.安裝所需Tool
```
apt-get update
apt-get install sshpass
apt install nfs-common -y
apt -y install open-iscsi multipath-tools -y
systemctl restart iscsid.service
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```


### 3.下載Unity CSI及CSI Snapshot
```
git clone https://github.com/dell/csi-unity.git
git clone https://github.com/kubernetes-csi/external-snapshotter/
```


### 4.安裝snapshotter
```
cd ~/external-snapshotter
kubectl kustomize client/config/crd | kubectl create -f -
kubectl -n kube-system kustomize deploy/kubernetes/snapshot-controller | kubectl create -f -
```


### 5.安裝CSI
```
kubectl create namespace unity
cd ~/csi-unity/dell-csi-helm-installer/
git clone https://github.com/dell/helm-charts.git
openssl s_client -showcerts -connect 172.22.46.231:443 </dev/null 2>/dev/null | openssl x509 -outform PEM > ca_cert_0.pem
kubectl create secret generic unity-certs-0 -n unity --from-file=cert-0=ca_cert_0.pem
```

### 6.建立secret.yaml

```
vi secret.yaml
```
* 輸入下列資訊
```
storageArrayList:
  - arrayId: "VIRT241266VLG2" # Unity Array ID
    username: "admin" # Unity User Name
    password: "XXXXXXXXXXX" # Unity Password
    endpoint: "https://172.22.46.231/" # Unity MGMT IP
    skipCertificateValidation: true
    isDefault: true
```

```
kubectl create secret generic unity-creds -n unity --from-file=config=secret.yaml
```
* 修改secret.yaml後更新
```
kubectl create secret generic unity-creds -n unity --from-file=config=secret.yaml -o yaml --dry-run | kubectl replace -f -
```

### 7.下載values.yaml

```
wget -O values.yaml https://raw.githubusercontent.com/dell/helm-charts/main/charts/csi-unity/values.yaml
./csi-install.sh --namespace unity --values ./values.yaml
```


### 8.修改values.yaml
* 若CSI版本不同，需修改 version

```
vi values.yaml
```
```
version: "v2.11.0"
```
* 若為需掛載不同網段，需由Storage進行設定

![](https://github.com/Andy0583/Dell-CSI-for-Powerstore/blob/main/image/011.png?raw=true)

### 8.安裝CSI
* 所有密碼都略過，直到出現下列訊息後按"y"
* Press 'y' to continue or any other key to exit:
```
./csi-install.sh --namespace unity --values ./values.yaml
```

* 移除CSI

```
./csi-uninstall.sh --namespace unity
```


### 9.檢查是否完成
```
Press 'y' to continue or any other key to exit: y
|
|- Installing Driver                                                Success
  |
  |--> Waiting for Deployment unity-controller to be ready          Success
  |
  |--> Waiting for DaemonSet unity-node to be ready                 Success
------------------------------------------------------
> Operation complete
------------------------------------------------------
```


### 10.驗證
* 驗證Pod數量及是否皆於"Running"狀態
```
root@k8s1:~/csi-unity/dell-csi-helm-installer# kubectl get pod -n unity
NAME                                READY   STATUS    RESTARTS   AGE
unity-controller-778d97dffc-2zm94   5/5     Running   0          89s
unity-controller-778d97dffc-bx9hd   5/5     Running   0          89s
unity-node-52wv4                    2/2     Running   0          89s
unity-node-5mk6v                    2/2     Running   0          89s
```
* 查看Storage Hosts是否出現二台Worker
  
![](https://github.com/Andy0583/Dell-CSI-for-Powerstore/blob/main/image/010.png?raw=true)
