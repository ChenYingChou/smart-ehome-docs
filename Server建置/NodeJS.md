NodeJS 建置
---

參考網站：https://www.phpini.com/linux/ubuntu-install-node-js

#### 執行安裝： 可以先到 [這裡檢查最新版本](https://github.com/nodesource/distributions/tree/master/deb)
```sh
curl -sL https://rpm.nodesource.com/setup_10.x | sudo bash -
sudo apt-get install nodejs
```

安裝全域套件可能發生 `gyp WARN EACCES` 禁止 root 存取，此時可用 `--unsafe-perm` 避免此錯誤訊息:
```sh
sudo npm install -g --unsafe-perm some-package
```
