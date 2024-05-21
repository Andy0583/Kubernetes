# 安裝K8S
by Andy Hsu

### 修改root密碼 / 關閉防火牆 / 開啟root SSH
```
sudo passwd root
sudo ufw disable
```
```
sudo vim /etc/ssh/sshd_config
```
> PermitRootLogin yes
```
sudo systemctl restart ssh
```

### 更新Ubuntu
```
apt update -y && apt upgrade -y
```

### 修改Hostname及Hosts (依據不同主機名稱修改)
```
hostnamectl set-hostname "k8s1.andy.com"
cat >> /etc/hosts << EOF
172.22.46.241 k8s1.andy.com
172.22.46.242 k8s2.andy.com
172.22.46.243 k8s3.andy.com
EOF
```

### 關閉Swap
```
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### 安裝K8S
```
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay

cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system

apt-get -y install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    apt-transport-https \
    gpg

mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null  
apt-get update

apt-get install containerd.io  
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
systemctl restart containerd
systemctl enable containerd

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.27/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.27/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-cache madison kubelet
apt install -y kubelet=1.27.10-1.1 kubeadm=1.27.10-1.1  kubectl=1.27.10-1.1
apt-mark hold kubelet kubeadm kubectl

modprobe br_netfilter
```

### K8S初始化（Only for Master）
```
kubeadm init --control-plane-endpoint="172.22.46.241" --pod-network-cidr=10.244.0.0/16
```

![](https://github.com/Andy0583/Dell-CSI-for-Powerstore/blob/main/image/001.png?raw=true)
> 紀錄上圖"Then you can join any number of worker nodes by running the following on each as root"資訊
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

wget https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f kube-flannel.yml
```

### Worker Node加入K8S Cluster（Only for Workter）
```
kubeadm join 172.22.46.241:6443 --token 0bxz18.pl91tl6wuovyi04i \
        --discovery-token-ca-cert-hash sha256:860dfa06a3f7614fa4fa404fa8647ae06ce001b52f2a469bc2c76d93cc59d174
```
> 加入Master產出的Join資訊

### 於Master檢查Node資訊，確認安裝成功
![](https://github.com/Andy0583/Dell-CSI-for-Powerstore/blob/main/image/002.png?raw=true)
