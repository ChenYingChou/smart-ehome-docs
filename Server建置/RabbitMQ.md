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

#### 安裝 RabbitMQ 開發函式庫 (librabbitmq-devel 供 C/C++ 用)
```sh
# 若有安裝 yum remi repository
yum install -y librabbitmq-last librabbitmq-last-devel
# 否則用標準的版本
yum install -y librabbitmq-last librabbitmq-devel
```

---

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

#### 啟用 Web 管理及 MQTT/webMQTT 協定
```sh
# RabbitMQ: tcp:5672, SSL:5671
rabbitmq-plugins enable rabbitmq_management       # tcp:15672, SSL:15671
rabbitmq-plugins enable rabbitmq_mqtt             # tcp:1883 , SSL:8883
rabbitmq-plugins enable rabbitmq_web_mqtt         # tcp:15675, SSL:15674
```

#### 下載 rabbitmqadmin
```sh
cd /usr/local/bin
wget http://localhost:15672/cli/rabbitmqadmin
chmod +x rabbitmqadmin
```

#### 設定管理者帳號
```sh
rabbitmqctl add_user admin **請換置成你的密碼**
rabbitmqctl set_user_tags admin administrator
rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
```

#### 啟用 Web 認證及授權
```sh
rabbitmq-plutins enable rabbitmq_auth_backend_http
cat > /etc/rabbitmq/rabbitmq.conf << _EOT_
# rabbitmq.conf

log.file.level = error

##在鏈中使用兩個後端：首先是內部，然後是HTTP
auth_backends.1 = internal
auth_backends.2 = http

##認證
##內置的機制是“普通”，
##'AMQPLAIN'和'EXTERNAL'可以通過添加其他機制
##插件。
##
##相關文檔指南：http：//rabbitmq.com/authentication.html。
##
auth_mechanisms.1 = AMQPLAIN
auth_mechanisms.2 = PLAIN

#auth_backends.2 = http
auth_http.http_method	= post
auth_http.user_path     = http://localhost:8001/auth/user
auth_http.vhost_path    = http://localhost:8001/auth/vhost
auth_http.resource_path = http://localhost:8001/auth/resource
auth_http.topic_path    = http://localhost:8001/auth/topic
_EOT_
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

* 編輯 /etc/rabbitmq/rabbitmq.conf，增加新的設定:
    ```conf
    ## Logging to rabbit amq.rabbitmq.log exchange (can be true or false)
    ##
    log.exchange = true

    ## Loglevel to log to amq.rabbitmq.log exchange
    ##
    log.exchange.level = info
    ```

* 再重啟服務: `service rabbitmq-server restart`


#### 建立相關目錄及權限設定
```sh
mkdir -p /data/mq-data/db
mkdir -p /data/mq-data/www/auth
# ... 建立 sqlite3 資料庫: /data/mq-data/db/oisp.db (如後所述)
# ... 建立 PHP Web 認證及授權程式: /data/www/auth/index.php (如後所述)
chown -R root:nginx /data/mq-data
chown -R apache:nginx /data/mq-data/db      # 允許執行期 php-fpm 讀寫 sqlite 資料庫
chown -R nginx:nginx /data/mq-data/www      # 允許執行期 php-fpm 讀取(執行) php 程式
chmod 750 /data/mq-data /data/mq-data/db /data/mq-data/www
```

#### 建立 sqlite3 資料庫
資料庫中密碼是採用 CRYPT_BLOWFISH 加密, PHP 函數為 `password_hash('密碼', PASSWORD_BCRYPT)`, 請將下面對應的密碼欄位 ('`$2y$10...`') 換置掉。

```sql
-- sqlite3 /data/mq-data/db/oisp.db << _EOT_
PRAGMA foreign_keys=OFF;
BEGIN TRANSACTION;
CREATE TABLE users (
    id varchar(16) not null primary key,
    pwd varchar(60) not null,   -- php: password_hash($password,PASSWORD_BCRYPT);
    token varchar(32) not null default '',
    email varchar(200) not null default '',
    mobile varchar(20) not null default '',
    policy varchar(200) not null default '',
    is_active bool not null default false
);
INSERT INTO "users" VALUES('admin','$2y$10$W0z1.TCjcROTAjqVp/0GwOfMn6DBKTTJ91p4sUjbq8YvJTTS./xC2','4077d9941ce44ff8a1b6cdb74b40c196','service@example.com',NULL,'management policymaker monitoring administrator','true');
INSERT INTO "users" VALUES('tony','$2y$10$NrWVLc.Rd791nPfGvZ1d2O23c1KNiXc3p9qTqI0Y3PZ/YCvXJu1PJ','c218eccf0c5349753fb07e5862204086','tonychen@example.com',NULL,'','false');
COMMIT;
-- _EOT_
```

#### PHP Web 認證及授權範例
請先依 [nginx-php7](https://github.com/ChenYingChou/smart-ehome-docs/blob/master/Server%E5%BB%BA%E7%BD%AE/nginx-php7.md) 說明建立 Web 及 PHP 執行環境。

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
