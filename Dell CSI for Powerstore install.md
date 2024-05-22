# 安裝CSI
<body><p align="right">== by Andy Hsu ==</p></body>

### 1.安裝所需Tool
```
apt-get update
apt-get install sshpass
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
apt install nfs-common -y
```

### 2.下載PowerStore CSI及CSI Snapshot
```
git clone https://github.com/dell/csi-powerstore.git
git clone https://github.com/kubernetes-csi/external-snapshotter/
```

### 3.安裝snapshotter
```
cd ~/external-snapshotter
kubectl kustomize client/config/crd | kubectl create -f -
kubectl -n kube-system kustomize deploy/kubernetes/snapshot-controller | kubectl create -f -
```

### 4.安裝CSI
```
kubectl create namespace ps
cd ~/csi-powerstore/dell-csi-helm-installer/
git clone https://github.com/dell/helm-charts.git
openssl s_client -showcerts -connect 172.22.33.20:443 </dev/null 2>/dev/null | openssl x509 -outform PEM > ca_cert_0.pem
kubectl create secret generic powerstore-certs-0 -n ps --from-file=cert-0=ca_cert_0.pem
```

### 5.建立secret.yaml
```
vim secret.yaml
```
>```
>arrays:
>  - endpoint: "https://172.22.33.20/api/rest"
>    globalID: "PS556db8bad5cd"
>    username: "admin"
>    password: "XXXXXXXXXXXXXX"
>    skipCertificateValidation: true
>    isDefault: true
>    blockProtocol: "auto"
>    nasName: "andy-nas"
>```
```
kubectl create secret generic powerstore-config -n ps --from-file=config=secret.yaml
```

### 6.下載values.yaml
```
wget -O my-powerstore-settings.yaml https://github.com/dell/helm-charts/raw/csi-powerstore-2.10.0/charts/csi-powerstore/values.yaml
mv my-powerstore-settings.yaml values.yaml
```

### 7.若NFS不同網段，於values.yaml修改
```
vi values.yaml
```
>```
> externalAccess: "192.168.131.0/24"
>```

### 8.安裝CSI
```
./csi-install.sh --namespace ps --values ./values.yaml
```
> 所有密碼都略過，直到出現下列訊息後按"y"<p></P>
> Press 'y' to continue or any other key to exit:

### 9.檢查安裝是否正確完成
```
Press 'y' to continue or any other key to exit: y
|
|- Installing Driver                                                Success
  |
  |--> Waiting for Deployment powerstore-controller to be ready     Success
  |
  |--> Waiting for DaemonSet powerstore-node to be ready            Success
------------------------------------------------------
> Operation complete
------------------------------------------------------
```

### 9.驗證Pod數量及是否皆於"Running"狀態
```
root@k8s1:~/csi-powerstore/dell-csi-helm-installer# kubectl get pod -n ps
NAME                                     READY   STATUS    RESTARTS        AGE
powerstore-controller-54cd5fccc4-2zm94   6/6     Running   1 (3m51s ago)   4m50s
powerstore-controller-54cd5fccc4-bx9hd   6/6     Running   1 (3m48s ago)   4m50s
powerstore-node-52wv4                    2/2     Running   0               4m50s
powerstore-node-5mk6v                    2/2     Running   0               4m50s
```
