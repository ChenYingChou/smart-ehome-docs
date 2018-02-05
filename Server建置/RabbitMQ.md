## RabbitMQ-Server 建置


### Raspberry Pi 安裝

參考網站：https://tecadmin.net/install-rabbitmq-server-on-ubuntu/

#### 以 root 權限執行安裝
```sh
echo 'deb http://www.rabbitmq.com/debian/ testing main' | tee /etc/apt/sources.list.d/rabbitmq.list
wget -O- https://www.rabbitmq.com/rabbitmq-release-signing-key.asc | apt-key add -
apt-get update
apt-get install rabbitmq-server
```

---

### CentOS 7 安裝

#### 先安裝 Erlang

* 參考網頁 [Zero-dependency Erlang RPM for RabbitMQ](https://github.com/rabbitmq/erlang-rpm)

* 以 root 權限執行安裝:
    ```sh
    cd /tmp
    wget -O erlang-20.2.2-1.el7.centos.x86_64.rpm "https://bintray.com/rabbitmq/rpm/download_file?file_path=erlang%2F20%2Fel%2F7%2Fx86_64%2Ferlang-20.2.2-1.el7.centos.x86_64.rpm"
    yum install -y erlang-20.2.2-1.el7.centos.x86_64.rpm
    # rm -f erlang-20.2.2-1.el7.centos.x86_64.rpm
    ```

#### 安裝 RabbitMQ

* 參考網頁 [Installing on RPM-based Linux](https://www.rabbitmq.com/install-rpm.html)

* 以 root 權限執行安裝:
    ```sh
    cd /tmp
    wget https://dl.bintray.com/rabbitmq/all/rabbitmq-server/3.7.2/rabbitmq-server-3.7.2-1.el7.noarch.rpm
    rpm --import https://www.rabbitmq.com/rabbitmq-release-signing-key.asc
    yum install -y rabbitmq-server-3.7.2-1.el7.noarch.rpm
    #rm -f rabbitmq-server-3.7.2-1.el7.noarch.rpm
    ```

#### 安裝 RabbitMQ 開發函式庫 (librabbitmq-devel 供 C/C++ 用)：
```sh
# 若有安裝 yum remi repository
yum install -y librabbitmq-last librabbitmq-last-devel
# 否則用標準的版本
yum install -y librabbitmq-last librabbitmq-devel
```

---

#### 設定系統開機自動啟動 RabbitMQ：
```sh
mkdir -p /etc/systemd/system/rabbitmq-server.service.d
cat > /etc/systemd/system/rabbitmq-server.service.d/open_files.conf << _EOT_
[Service]
LimitNOFILE=16384
_EOT_
systemctl daemon-reload
systemctl enable rabbitmq-server
systemctl start rabbitmq-server
```

#### 啟用 Web 管理及 MQTT/webMQTT 協定：
```sh
# RabbitMQ: tcp:5672, SSL:5671
rabbitmq-plugins enable rabbitmq_management       # tcp:15672, SSL:15671
rabbitmq-plugins enable rabbitmq_mqtt             # tcp:1883 , SSL:8883
rabbitmq-plugins enable rabbitmq_web_mqtt         # tcp:15675, SSL:15674
```

#### 下載 rabbitmqadmin：
```sh
cd /usr/local/bin
wget http://localhost:15672/cli/rabbitmqadmin
chmod +x rabbitmqadmin
```

#### 設定管理者帳號：
```sh
rabbitmqctl add_user admin **admin-password**
rabbitmqctl set_user_tags admin administrator
rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
```
