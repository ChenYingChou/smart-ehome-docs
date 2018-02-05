nginx & php7 建置
---

參考網站：https://pimylifeup.com/raspberry-pi-nginx/

#### 以 root 權限執行安裝 nginx：
```sh
apt-get update
apt-get upgrade
apt-get remove apache2
apt-get install nginx
systemctl start nginx
```

#### 以 root 權限執行安裝 php7：
```sh
echo "deb http://mirrordirector.raspbian.org/raspbian/ stretch main contrib non-free rpi" >> /etc/apt/sources.list
cat >> /etc/apt/preferences <<< _EOT_

Package: *
Pin: release n=jessie
Pin-Priority: 600
_EOT_
apt-get update
apt-get install -t stretch php7.0-fpm
```

#### 擴充系統預設 I/O handles
```sh
cat > /etc/security/limits.d/99-nofile.conf << _EOT_
*               hard    nofile          51200
*               soft    nofile          51200
_EOT_
```