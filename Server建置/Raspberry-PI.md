<h2>Raspberry PI</h2>

[[TOC]]

### 安裝 Raspbian Stretch
1. 下載網頁: [Raspbian](https://www.raspberrypi.org/downloads/raspbian/)
1. 選擇 `Raspbian Stretch with desktop` 下載
1. 參見 [安裝說明](https://www.raspberrypi.org/documentation/installation/installing-images/README.md)

### 安裝 NodeJS
由於 Raspbian Stretch (2018/11) 所提供 nodejs v11 並無 npm 套件，因此必須改用下列方式安裝

參考網頁來源: [NodeJs-Raspberry-Pi](https://github.com/audstanley/NodeJs-Raspberry-Pi/blob/master/README.md)

```sh
curl -sL\
 https://raw.githubusercontent.com/audstanley/NodeJs-Raspberry-Pi/master/Install-Node.sh \
| sudo bash

# 將 /usr/lib/node_modules 指到正確的位置
sudo ln -nfs /opt/nodejs/lib/node_modules /usr/lib/

# 檢視安裝的版本
node -v

## 下行指令可選擇指定版本 10 (LTS)
# sudo node-install -v 10
## then you will get prompted with which
## specific version of 10 you wish to install

sudo npm install -g --unsafe-perm axios commander dateformat deep-diff mqtt \
    bcryptjs sqlite3
```

### 安裝 Web Server nginx 及 php

參考 [nginx & php7 建置](./nginx-php7.md)

### 安裝 RabbitMQ

參考 [RabbitMQ 建置](./RabbitMQ.md)

### 有線網卡 eth0 固定 IP

```sh
echo ">>> Patch /usr/lib/dhcpcd5/dhcpcd"
patch -l -s -N -r - /usr/lib/dhcpcd5/dhcpcd << '_EOT_'
--- dhcpcd~	2019-03-17 08:19:13.990370181 +0800
+++ dhcpcd	2019-03-17 08:19:44.580063877 +0800
@@ -3,12 +3,12 @@
 DHCPCD=/sbin/dhcpcd
 INTERFACES=/etc/network/interfaces

-if grep -q -E "^[[:space:]]*iface[[:space:]]*.*[[:space:]]*inet[[:space:]]*(dhcp|static)" \
-		$INTERFACES; then
-	echo "Not running dhcpcd because $INTERFACES"
-	echo "defines some interfaces that will use a"
-	echo "DHCP client or static address"
-	exit 6
-fi
+#if grep -q -E "^[[:space:]]*iface[[:space:]]*.*[[:space:]]*inet[[:space:]]*(dhcp|static)" \
+#		$INTERFACES; then
+#	echo "Not running dhcpcd because $INTERFACES"
+#	echo "defines some interfaces that will use a"
+#	echo "DHCP client or static address"
+#	exit 6
+#fi

 exec $DHCPCD $@
_EOT_

echo ">>> Patch /etc/network/interfaces"
patch -l -s -N -r - /etc/network/interfaces << '_EOT_'
--- interfaces~	2019-03-17 08:23:50.379337594 +0800
+++ interfaces	2019-03-17 08:25:58.158151092 +0800
@@ -5,3 +5,16 @@

 # Include files from /etc/network/interfaces.d:
 source-directory /etc/network/interfaces.d
+
+auto lo
+iface lo inet loopback
+
+auto eth0
+iface eth0 inet static
+    address 192.168.1.12
+    netmask 255.255.255.0
+    network 192.168.1.0
+    gateway 192.168.1.254
+    dns-nameserver 192.168.1.254
+    #dns-search amma.com.tw
+
_EOT_

echo ">>> Deny eth0 in /etc/dhcpcd.conf"
echo "denyinterfaces eth0" >> /etc/dhcpcd.conf
```
