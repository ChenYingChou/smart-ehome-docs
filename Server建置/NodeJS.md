NodeJS 建置
---

參考網站：[ubuntu install node-js](https://www.phpini.com/linux/ubuntu-install-node-js)

### 執行安裝： 可以先到 [這裡檢查最新版本](https://github.com/nodesource/distributions/tree/master/deb)
```sh
curl -sL https://rpm.nodesource.com/setup_10.x | sudo bash -
sudo apt-get install nodejs
```

### 樹莓派 (Raspbian Stretch)
由於 Raspbian Stretch (2018/11) 所提供 nodejs v11 並無 npm 套件，因此必須改用下列方式安裝

#### Raspbian Stretch ARM 安裝
參考網頁: [NodeJs Raspberry Pi](https://github.com/audstanley/NodeJs-Raspberry-Pi/blob/master/README.md)
```sh
wget -O - https://raw.githubusercontent.com/audstanley/NodeJs-Raspberry-Pi/master/Install-Node.sh | sudo bash

# 指定版本 10 (LTS)
sudo node-install -v 10
# then you will get prompted with which
# specific version of 10 you wish to install

ln -nfs /opt/nodejs/lib/node_modules /usr/lib/
```

#### Raspbian Stretch Desktop 安裝
參考網頁: [install latest nodejs npm on debian](https://tecadmin.net/install-latest-nodejs-npm-on-debian/)

```sh
sudo apt-get install -y curl software-properties-common
curl -sL https://deb.nodesource.com/setup_10.x | sudo bash -
apt-get install -y nodejs
# 只支援到 nodejs v8
node -v
```

### npm 安裝套件注意事項
安裝全域套件可能發生 `gyp WARN EACCES` 禁止 root 存取，此時可用 `--unsafe-perm` 避免此錯誤訊息:
```sh
sudo npm install -g --unsafe-perm some-package
# 或將之設定到 /root/.npmrc
sudo sh -c 'echo "unsafe-perm=true" > ~/.npmrc'
```
