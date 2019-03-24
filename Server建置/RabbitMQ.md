<h2>RabbitMQ-Server 建置</h2>

```sh
# 以下所有安裝或設定請用 root 身份執行
sudo -i
```
[[TOC]]

### Raspberry Pi 安裝

參考網站:
* [Erlang/OTP packages for Debian and Ubuntu](https://github.com/rabbitmq/erlang-debian-package/)
* [Installing on Debian and Ubuntu](https://www.rabbitmq.com/install-debian.html)

```sh
apt-key adv --keyserver "hkps.pool.sks-keyservers.net" --recv-keys "0x6B73A36E6026DFCA"
wget -O - "https://github.com/rabbitmq/signing-keys/releases/download/2.0/rabbitmq-release-signing-key.asc" | sudo apt-key add -
wget -O - 'https://dl.bintray.com/rabbitmq/Keys/rabbitmq-release-signing-key.asc' | sudo apt-key add -
apt-get install -y apt-transport-https

cat > /etc/apt/sources.list.d/bintray.rabbitmq.list << '_EOT_'
# This repository provides RabbitMQ packages
# See below for supported distribution and component values
deb https://dl.bintray.com/rabbitmq-erlang/debian stretch erlang
deb https://dl.bintray.com/rabbitmq/debian stretch main
_EOT_

cat > /etc/apt/preferences.d/erlang << '_EOT_'
# /etc/apt/preferences.d/erlang
Package: erlang*
Pin: release o=Bintray
#Pin: version 1:21.2.5-1
Pin-Priority: 1000
_EOT_

apt-cache policy
apt-get update
apt-get install -t buster -y erlang-nox rabbitmq-server
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

#### 安裝 RabbitMQ 開發函式庫 (librabbitmq-devel 供 C/C++ 用)
```sh
# 若有安裝 yum remi repository
yum install -y librabbitmq-last librabbitmq-last-devel
# 否則用標準的版本
yum install -y librabbitmq-last librabbitmq-devel
```

### RabbitMQ 設定及更新

#### 設定系統開機自動啟動 RabbitMQ
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

#### 啟用 WebAdmin/MQTT/webMQTT 及基本帳號權限
```sh
#!/bin/bash

read_password() {
    [ -z "$1" ] && return
    local pwdsz=$(shuf -i 12-30 -n 1)
    local password
    echo ""
    echo -n "Enter password for $1: "
    read password
    [ -z "${password}" ] && password=$(< /dev/urandom tr -dc _A-Za-z0-9- | head -c${pwdsz})
    [ "$(expr substr "$password" 1 1)" = "-" ] && password="x${password}"
    eval $1Pwd="$password"
}

echo "RabbitMQ Setup ..."

[ -f ~/.rabbitmq.pwd ] && . ~/.rabbitmq.pwd
echo -e "\n>>> Remove old users ... please ignore the errors"
rabbitmqctl delete_user admin
rabbitmqctl delete_user oisp

[ -z "${adminPwd}" ] && read_password 'admin'
[ -z "${oispPwd}"  ] && read_password 'oisp'
cat > ~/.rabbitmq.pwd << _EOT_
adminPwd="${adminPwd}"
oispPwd="${oispPwd}"
_EOT_

[ -f ~/.rabbitmqadmin.conf ] && mv ~/.rabbitmqadmin.conf ~/.rabbitmqadmin.conf.bak

echo -e "\n>>> Create ~/.rabbitmqadmin.conf"
cat > ~/.rabbitmqadmin.conf << _EOT_
# rabbitmqadmin.conf START

[default]
hostname = localhost
port = 15672
username = admin
password = ${adminPwd}
#vhost = /
#ssl = True

[oisp]
hostname = localhost
port = 15672
username = oisp
password = ${oispPwd}
vhost = oisp
#ssl = True

# rabbitmqadmin.conf END
_EOT_

chmod go-rwx ~/.rabbitmq*

echo -e "\n>>> Enable rabbitmq plugins: web-management, mqtt, web-mqtt, web-auth"
# RabbitMQ: amqp            # tcp:5672 , SSL:5671
# rabbitmq_management       # tcp:15672, SSL:15671
# rabbitmq_mqtt             # tcp:1883 , SSL:8883
# rabbitmq_web_mqtt         # tcp:15675, SSL:15674
rabbitmq-plugins enable \
	rabbitmq_amqp1_0 \
	rabbitmq_auth_backend_cache \
	rabbitmq_auth_backend_http \
	rabbitmq_management \
	rabbitmq_mqtt \
	rabbitmq_web_mqtt

echo -e "\n>>> Create vhost: oisp"
rabbitmqctl add_vhost oisp

echo -e "\n>>> Create user: admin for vhost / and oisp"
rabbitmqctl add_user admin "${adminPwd}"
rabbitmqctl set_user_tags admin administrator
rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
rabbitmqctl set_permissions -p oisp admin ".*" ".*" ".*"

echo -e "\n>>> Create user oisp for vhost oisp"
rabbitmqctl add_user oisp "${oispPwd}"
rabbitmqctl set_user_tags oisp administrator
rabbitmqctl set_permissions -p oisp oisp  ".*" ".*" ".*"

if [ ! -x /usr/bin/rabbitmqadmin ]; then
    echo -e "\n>>> Download rabbitmqadmin"
    wget -O /usr/bin/rabbitmqadmin http://localhost:15672/cli/rabbitmqadmin
    chmod +x /usr/bin/rabbitmqadmin
    patch -s -N -r - /usr/bin/rabbitmqadmin << '_EOT_'
--- rabbitmqadmin-3.7.13~	2019-03-21 14:02:15.209366755 +0800
+++ rabbitmqadmin-3.7.13	2019-03-21 14:03:13.225569011 +0800
@@ -25,6 +25,7 @@
 import socket
 import ssl
 import traceback
+import time

 try:
     from signal import signal, SIGPIPE, SIG_DFL
@@ -71,7 +72,7 @@

 VERSION = '3.7.13'

-LISTABLE = {'connections': {'vhost': False, 'cols': ['name', 'user', 'channels']},
+LISTABLE = {'connections': {'vhost': False, 'cols': ['name', 'user', 'connected_at']},
             'channels':    {'vhost': False, 'cols': ['name', 'user']},
             'consumers':   {'vhost': True},
             'exchanges':   {'vhost': True,  'cols': ['name', 'type']},
@@ -500,7 +501,9 @@


 def maybe_utf8(s):
-    if isinstance(s, int):
+    if isinstance(s, int) or isinstance(s, long):
+        if s >= 1514764800000: # 2018-01-01T00:00:00.000Z
+            return time.strftime("%Y-%m-%d %H:%M:%S", time.localtime(s/1000))
         # s can be also an int for ex messages count
         return str(s)
     if isinstance(s, float):
_EOT_
fi

#rabbitmqctl add_user user1 user1.password
#rabbitmqctl set_permissions -p oisp user1 "^mqtt\.from\..*" "^mqtt\.to" "^mqtt\.from"

echo -e "\n>>> Create exchanges mqtt and bcast for vhost oisp"
rabbitmqadmin -V oisp declare exchange name=mqtt type=topic
rabbitmqadmin -V oisp declare exchange name=bcast type=fanout

#rabbitmqadmin -N oisp declare binding source=mqtt destination=bcast routing_key="#" destination_type=exchange

echo -e "\n>>> Create policy for vhost oisp"
#
# Create policy for MQTT
#
rabbitmqctl set_policy -p oisp osvc "^mqtt-subscription-2:" \
    '{"expires":259200000, "max-length-bytes":102400, "message-ttl":30000}' \
    --priority 1 \
    --apply-to queues

rabbitmqctl set_policy -p oisp ouser "^mqtt-subscription-1:" \
    '{"expires":150000, "max-length":500, "max-length-bytes":102400}' \
    --priority 1 \
    --apply-to queues

echo -e "\n==================="
echo " RMQ administrator: admin, Password: ${adminPwd}"
echo "vhost oisp manager: oisp , Password: ${oispPwd}"

if [ ! -f /etc/rabbitmq/rabbitmq.conf ]; then
    # 啟用 Web 認證及授權
    cat > /etc/rabbitmq/rabbitmq.conf << '_EOT_'
# rabbitmq.conf

## https://www.rabbitmq.com/logging.html
log.file.level = error
#log.connection.level = info

## Logging to rabbit amq.rabbitmq.log exchange (can be true or false)
#log.exchange = true

## Loglevel to log to amq.rabbitmq.log exchange
#log.exchange.level = info

## 以上 log 可用下列 event 來取代: https://www.rabbitmq.com/event-exchange.html
##   rabbitmq-plugins enable rabbitmq_event_exchange

## define multiple listeners using listener names
#listeners.tcp.default = 5672
#listeners.tcp.other_port = 5673
#listeners.tcp.other_ip = x.x.x.x
#listeners.tcp.other2_port = 5674
#listeners.tcp.other2_ip = x.x.x.x
## <-->
#listeners.tcp.1 = :::5672
#listeners.tcp.2 = :::5673
#listeners.tcp.3 = :::5674

#[ssl]# listeners.ssl.default            = 5671
#[ssl]# ssl_options.cacertfile           = /etc/ssl/your-domain/your-domain-www.pem
#[ssl]# ssl_options.certfile             = /etc/ssl/your-domain/your-domain-www.crt
#[ssl]# ssl_options.keyfile              = /etc/ssl/your-domain/your-domain.key
#[ssl]# ssl_options.versions.1           = tlsv1.2
#[ssl]# ssl_options.versions.2           = tlsv1.1
#[ssl]# ssl_options.verify               = verify_peer
#[ssl]# ssl_options.fail_if_no_peer_cert = false

#[ssl]# ## Web Management SSL
#[ssl]# management.listener.port = 15671
#[ssl]# management.listener.ssl  = true
#[ssl]# management.listener.ssl_opts.cacertfile           = /etc/ssl/your-domain/your-domain-www.pem
#[ssl]# management.listener.ssl_opts.certfile             = /etc/ssl/your-domain/your-domain-www.crt
#[ssl]# management.listener.ssl_opts.keyfile              = /etc/ssl/your-domain/your-domain.key
#[ssl]# management.listener.ssl_opts.versions.1           = tlsv1.2
#[ssl]# management.listener.ssl_opts.versions.2           = tlsv1.1
#[ssl]# management.listener.ssl_opts.verify               = verify_none
#[ssl]# management.listener.ssl_opts.fail_if_no_peer_cert = false

## 在鏈中使用多個後端：internal, cache(http)
auth_backends.1 = internal
auth_backends.2 = cache
#auth_backends.3 = http

## 認證相關文檔指南: http://rabbitmq.com/authentication.html
auth_mechanisms.1 = AMQPLAIN
auth_mechanisms.2 = PLAIN

## Cache setup
auth_cache.cached_backend = http
auth_cache.cache_ttl = 1800000          # milliseconds

auth_http.http_method   = post
auth_http.user_path     = http://localhost:8001/auth/user
auth_http.vhost_path    = http://localhost:8001/auth/vhost
auth_http.resource_path = http://localhost:8001/auth/resource
auth_http.topic_path    = http://localhost:8001/auth/topic

#########
## MQTT # https://www.rabbitmq.com/mqtt.html
#########
mqtt.allow_anonymous  = false
mqtt.vhost            = oisp
mqtt.exchange         = mqtt

## 24 hours by default
#-# mqtt.subscription_ttl = 86400000
#-# mqtt.prefetch         = 10
#-# mqtt.listeners.ssl    = none

#[ssl]# mqtt.listeners.ssl.default = 8883
mqtt.listeners.tcp.default = 1883
#-# mqtt.tcp_listen_options.backlog = 128
#-# mqtt.tcp_listen_options.nodelay = true

## define multiple listeners using listener names
#-# mqtt.listeners.tcp.default = 1883
#-# #mqtt.listeners.tcp.other2_port = 1884
#-# #mqtt.listeners.tcp.other3_port = 1885
## <-->
#-# mqtt.listeners.tcp.1 = :::1883
#-# mqtt.listeners.tcp.2 = :::1884
#-# mqtt.listeners.tcp.3 = :::1885

#############
## Web-MQTT #
#############
web_mqtt.tcp.port       = 15675
#[ssl]# web_mqtt.ssl.port       = 15674
#[ssl]# web_mqtt.ssl.backlog    = 1024
#[ssl]# web_mqtt.ssl.cacertfile = /etc/ssl/your-domain/your-domain-www.pem
#[ssl]# web_mqtt.ssl.certfile   = /etc/ssl/your-domain/your-domain-www.crt
#[ssl]# web_mqtt.ssl.keyfile    = /etc/ssl/your-domain/your-domain.key
#[ssl]# #web_mqtt.ssl.password   = changeme # needed when private key has a passphrase
_EOT_

    echo -e "\nRestart rabbitmq-server ..."
    service rabbitmq-server restart
fi
```

#### 每次更新版本
* 基本 schema 可能有異動需手動更新:
    ```sh
    cd /var/lib/rabbitmq/schema
    cp -af rabbit.schema /tmp/
    cp -af /usr/lib/rabbitmq/lib/rabbitmq_server-<新版號>/priv/schema/rabbit.schema .
    chown rabbitmq:rabbitmq rabbit.schema
    chmod o-rwx rabbit.schema
    ```
* 或編輯 /etc/rabbitmq/rabbitmq.conf，增加新的設定
* 重啟服務: `service rabbitmq-server restart`

### OISP 初始化目錄及設定

#### 建立目錄及權限設定
```sh
apt-get install -y sqlite3

umask 0022
mkdir -p /opt/oisp
ln -nfs /opt/oisp /
cd /opt/oisp
mkdir -p api auth cache db lib modules modules/sys sys udpsvr www www/api www/auth
# ... 建立 sqlite3 資料庫: /oisp/db/oisp.db (如後所述)
# ... 建立 PHP Web 認證及授權程式: /oisp/www/auth/index.php (如後所述)

chown -R pi:pi /opt/oisp      # 允許執行期 php-fpm 讀寫 sqlite 資料庫
chmod 751 /opt/oisp
cd /opt/oisp
chmod 755 *
chmod o-rwx modules sys udpsvr
chmod g+rwx db cache
chgrp www-data db cache
chown www-data:pi db/*.db
chmod g+w db/*.db
find . -name '*.json' | xargs -r chmod g+w

RMQ_AUTH="/etc/nginx/sites-available/rabbitmq-auth"
if [ ! -f ${RMQ_AUTH} ]; then
    cat > ${RMQ_AUTH} << '_EOT_'
# RabbitMQ authentication & authorisation
server {
    listen 127.0.0.1:8001 default_server;

    access_log off;
   #error_log off;

    location /auth {
        root        /oisp/www;
        index       index.php;
        fastcgi_split_path_info ^(/auth(?:/.+?\.php)?)($|/.*$);
        try_files   $uri $uri/ /auth/index.php$fastcgi_path_info?$query_string;
        include     php.conf;
    }
}
_EOT_
    ln -nfs ${RMQ_AUTH} /etc/nginx/sites-enabled/
    # 重啟 nginx 服務
    nginx -t && service nginx reload
fi
```

#### 建立 sqlite3 資料庫
資料庫中密碼是採用 CRYPT_BLOWFISH 加密, PHP 函數為 `password_hash('密碼', PASSWORD_BCRYPT)`, 請將下面對應的密碼欄位 ('`$2y$10...`') 換置掉。

```sql
--# sqlite3 /data/mq-data/db/oisp.db << '_EOT_'
PRAGMA foreign_keys=OFF;
BEGIN TRANSACTION;

CREATE TABLE IF NOT EXISTS "auth_code" (
    `code`      text NOT NULL,
    `at_time`   text NOT NULL,
    PRIMARY KEY(`code`)
);

CREATE TABLE IF NOT EXISTS "reg_users_map" (
    `userid`    text NOT NULL,
    `device`    text NOT NULL,
    `id`        text NOT NULL UNIQUE,       -- 5 bytes: A~E x 10^4
    PRIMARY KEY(`userid`, `device`)
);

CREATE TABLE IF NOT EXISTS "reg_users" (
    `userid`    text NOT NULL,
    `name`      text NOT NULL,
    `is_admin`  int NOT NULL DEFAULT 0,     -- admin for 1:reg_users, 2:module
    `reg_time`  text NOT NULL DEFAULT '',
    `uid`       text NOT NULL UNIQUE,       -- 4 bytes (62|64)^4
    PRIMARY KEY(`userid`)
);

CREATE TABLE IF NOT EXISTS "users" (
    `id`        text NOT NULL,
    `pwd`       text NOT NULL,   -- php: password_hash($password,PASSWORD_BCRYPT);
    `email`     text NOT NULL DEFAULT '',
    `mobile`    text NOT NULL DEFAULT '',
    `token`     text NOT NULL DEFAULT '',
    `policy`    text NOT NULL DEFAULT '',
    `lang`      text NOT NULL DEFAULT 'en',
    `is_active` int NOT NULL DEFAULT 1,
    `type`      int NOT NULL DEFAULT 0,     -- 0:system, 1:reg_users, 2:module
    `uid`       text NOT NULL DEFAULT '',   -- copy from reg_users.uid if type >= 1
    `os`        text NOT NULL DEFAULT '',   -- 'iOS', 'android', 'web'
    PRIMARY KEY(`id`)
);

-- INSERT INTO `reg_users` VALUES('admin', 'Administrator', 2, '', 'admin');
-- INSERT INTO `users` VALUES('admin','$2y$10$enWbeal3axm.KP15U.cr.xxxxxxxxxxxx/T9MCGl2xxxxxxxxxxxx','service@amma.com.tw','','','management policymaker monitoring administrator','tw',1,0,'','');
COMMIT;
--# _EOT_
```

### PHP Web 認證及授權範例
請先依 [nginx-php7](./nginx-php7.md) 說明建立 Web 及 PHP 執行環境。

```php
<?php  # /data/mq-data/www/auth/index.php

$dbdns = 'sqlite:/data/mq-data/db/oisp.db';
$logfile = '/data/mq-data/db/log';

function parse_path_info()
{
    $info = [];
    if (array_key_exists('PATH_INFO', $_SERVER)) $s = $_SERVER['PATH_INFO'];
    if (empty($s)) $s = $_SERVER['REQUEST_URI'];
    if (!empty($s)) {
        $path = explode('/', $s);
        foreach ($path as $x) {
            if (!empty($x) && strtolower(substr($x, -4)) !== '.php') {
                @list($key, $value) = explode('=', $x, 2);
                $info[$key] = $value;
            }
        }
    }
    return $info;
}

function my_log($msg)
{
    global $logfile;
    $dt = new DateTime();
    error_log($dt->format('m-d H:i:s.v')." $msg\n", 3, $logfile);
}

$info = parse_path_info();
$uri = json_encode($info, JSON_UNESCAPED_SLASHES|JSON_UNESCAPED_UNICODE);
$req = json_encode($_REQUEST, JSON_UNESCAPED_SLASHES|JSON_UNESCAPED_UNICODE);
my_log("URI=$uri : $req");

$response = 'deny';
if (array_key_exists('username', $_REQUEST)) {
    $username = $_REQUEST['username'];
    try {
        $opts = [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION];
        $db = new PDO($dbdns, null, null, $opts);

        $stm = $db->prepare('select * from users where id = ?');
        $stm->execute([$username]);
        $user = $stm->fetch(PDO::FETCH_ASSOC);
        if (!empty($user)) {
            foreach($info as $key => $value) {
                if (is_array($value)) $value = implode(',', $value);
                switch ($key) {
                    case 'user':	// username, password
                        // allow management policymaker monitoring administrator
                        $password = @$_REQUEST['password'];
                        if (password_verify($password, $user['pwd'])) {
                            $response = trim("allow {$user['policy']}");
                        } else {
                            $response = 'deny';
                        }
                        my_log("Response=$response");
                        $stm = null;
                        break;
                    case 'vhost':	// username, vhost, ip(client)
                    case 'resource':// username, vhost, resource(exchange, queue, topic),
                        // name(resource), permission(configure, write, read)
                    case 'topic':	// username, vhost, resource(topic), name(exchange),
                        // permission(write, read), routing_key
                        $response = 'allow';
                        break;
                    case '_': // jQuery adding to prevent server cached page
                        break;
                }
            }
        }
    } catch(Exception $e) {
        my_log('Exception: '.$e->getMessage());
    } finally {
        $db = null;
    }
}

header('Cache-Control: max-age=3600');
echo $response;
```
