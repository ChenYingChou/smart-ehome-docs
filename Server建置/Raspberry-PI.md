<h2>Raspberry PI</h2>

[[TOC]]

### 安裝 Raspbian Stretch
1. 下載網頁: [Raspbian](https://www.raspberrypi.org/downloads/raspbian/)
1. 選擇 `Raspbian Stretch with desktop` 下載
1. 參見 [安裝說明](https://www.raspberrypi.org/documentation/installation/installing-images/README.md)

### 系統更新

```sh
sudo apt-get update
sudo apt-get upgrade -y

# 參考網頁 https://unix.stackexchange.com/questions/442698/when-i-log-in-it-hangs-until-crng-init-done
# 2019-03-24: Linux 4.14~4.16 在未接鍵盤時, 系統開機會在 crng init 停滯許久
# 安裝 rng-tools 套件可解決此問題
sudo apt-get install rng-tools
```

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
    bcryptjs sqlite3 amqplib koa koa-bodyparser koa-router serialport
```

### 安裝 Web Server nginx 及 php

參考 [nginx & php7 建置](./nginx-php7.md)

### 安裝 RabbitMQ

參考 [RabbitMQ 建置](./RabbitMQ.md)

### 有線網卡 eth0 固定 IP

 1. 請儘量使用原系統配置的 dhcpcd 功能，避免使用此方法。
 1. 以 root 身份 (`sudo -i`) 執行下列步驟，事前做好檔案備份:
    * ~~檔案 `/usr/lib/dhcpcd5/dhcpcd`: 註解掉 `if ... fi` 共七行~~ (已不用)
    * 檔案 `/etc/dhcpcd.conf`: 增加 `denyinterfaces eth0` 於最後一行
    * 檔案 `/etc/network/interfaces.d/`: 此目錄原本為空的，增加一個檔案 `eth0`。請將 `address` / `netmask` / `network` / `gateway` / `dns-nameserver` 換成正確的設定值。

```sh
if [ ! -f /etc/network/interfaces.d/eth0 ]; then
    echo ">>> Create /etc/network/interfaces.d/eth0"
    cat > /etc/network/interfaces.d/eth0 << '_EOT_'
auto eth0
iface eth0 inet static
    address 192.168.1.12
    netmask 255.255.255.0
    network 192.168.1.0
    gateway 192.168.1.1
    dns-nameserver 192.168.1.1
    #dns-search smart-ehome.com
_EOT_
    echo ">>> Deny eth0 in /etc/dhcpcd.conf"
    echo "denyinterfaces eth0" >> /etc/dhcpcd.conf
fi
```
