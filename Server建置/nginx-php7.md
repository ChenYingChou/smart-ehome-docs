nginx & php7 建置
---

參考網站：https://pimylifeup.com/raspberry-pi-nginx/

#### 以 root 權限執行安裝 nginx：
````
apt-get update
apt-get upgrade
apt-get remove apache2
apt-get install nginx
systemctl start nginx
````

#### 以 root 權限執行安裝 php7：
````
echo "deb http://mirrordirector.raspbian.org/raspbian/ stretch main contrib non-free rpi" >> /etc/apt/sources.list
cat >> /etc/apt/preferences <<< _EOT_

Package: *
Pin: release n=jessie
Pin-Priority: 600
_EOT_
apt-get update
apt-get install -t stretch php7.0-fpm
````
