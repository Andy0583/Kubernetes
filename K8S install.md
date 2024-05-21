# 安裝K8S
### 修改root密碼 / 關閉防火牆 / 開啟root SSH
```
sudo passwd root
sudo ufw disable
sudo vim /etc/ssh/sshd_config
    [PermitRootLogin yes]
sudo systemctl restart ssh
```

### 更新Ubuntu
```
apt update -y && apt upgrade -y
```

