## Raspberry PI

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
ln -nfs /opt/nodejs/lib/node_modules /usr/lib/

# 檢視安裝的版本
node -v

## 下行指令可選擇指定版本 10 (LTS)
# sudo node-install -v 10
## then you will get prompted with which
## specific version of 10 you wish to install

sudo npm install -g --unsafe-perm axios commander dateformat deep-diff mqtt
```

### 安裝 Web Server nginx 及 php

參考 [nginx & php7 建置](./nginx-php7.md)

### 安裝 RabbitMQ

參考 [RabbitMQ 建置](./RabbitMQ.md)
