# Harbor架設
### 1.前置作業
* root密碼修改 / 啟用
```
sudo passwd root
sudo ufw disable
sudo vi /etc/ssh/sshd_config
```
> ```
> PermitRootLogin yes
> ```
```
sudo systemctl restart ssh
```

* 更新Ubuntu
```
apt update -y && apt upgrade -y
```


### 2.安裝Docker
```
curl -fsSL https://get.docker.com | sh
systemctl start docker && systemctl enable docker
apt install docker-compose -y
```


### 3.下載Harbor及設定
```
wget https://github.com/goharbor/harbor/releases/download/v2.10.2/harbor-offline-installer-v2.10.2.tgz
tar xf harbor-offline-installer-v2.10.2.tgz -C /usr/local/src/
mkdir /usr/local/src/harbor/certs
cd /usr/local/src/harbor/certs
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -sha512 -days 3650 -subj "/C=CN/ST=TP/L=TP/O=SmartX/OU=Lab/CN=harbor.andy.com" -key ca.key -out ca.crt
openssl genrsa -out harbor.andy.com.key 4096
openssl req -sha512 -new -subj "/C=CN/ST=TP/L=TP/O=SmartX/OU=Lab/CN=harbor.andy.com" -key harbor.andy.com.key -out harbor.andy.com.csr
```
```
vi v3.ext
```
```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
 
[alt_names]
DNS.1=harbor.andy.com
DNS.2=andy.com
DNS.3=harbor
```
```
openssl x509 -req -sha512 -days 3650 -extfile v3.ext -CA ca.crt -CAkey ca.key -CAcreateserial -in harbor.andy.com.csr -out harbor.andy.com.crt
openssl x509 -inform PEM -in harbor.andy.com.crt -out harbor.andy.com.cert
cd ..
cp harbor.yml.tmpl harbor.yml
```
```
vi harbor.yml
```
 >````
> hostname: harbor.andy.com
> http:
>   port: 9001
> https:
>   port: 443
>   certificate: /usr/local/src/harbor/certs/harbor.andy.com.cert
>   private_key: /usr/local/src/harbor/certs/harbor.andy.com.key
> harbor_admin_password: XXXXXXXXXXX
> ```


### 4.進行初始安裝
```
./install.sh
```

* 安裝完成畫面
```
[+] Running 10/10
 ✔ Network harbor_harbor        Created                                                                                              0.1s
 ✔ Container harbor-log         Started                                                                                              1.0s
 ✔ Container registryctl        Started                                                                                              1.6s
 ✔ Container harbor-db          Started                                                                                              1.6s
 ✔ Container registry           Started                                                                                              1.6s
 ✔ Container harbor-portal      Started                                                                                              1.4s
 ✔ Container redis              Started                                                                                              1.4s
 ✔ Container harbor-core        Started                                                                                              1.8s
 ✔ Container nginx              Started                                                                                              2.1s
 ✔ Container harbor-jobservice  Started                                                                                              2.1s
✔ ----Harbor has been installed and started successfully.----
```

```
vi /etc/docker/daemon.json
```
```
{
"insecure-registries" : ["harbor.andy.com:443", "0.0.0.0/0"]
}
```
```
docker-compose down -v
systemctl daemon-reload
systemctl restart docker
docker-compose up -d
```

