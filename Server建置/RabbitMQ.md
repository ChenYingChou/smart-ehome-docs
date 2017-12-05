RabbitMQ-Server 建置
---

參考網站：https://tecadmin.net/install-rabbitmq-server-on-ubuntu/

#### 以 root 權限執行安裝：
````
echo 'deb http://www.rabbitmq.com/debian/ testing main' | tee /etc/apt/sources.list.d/rabbitmq.list
wget -O- https://www.rabbitmq.com/rabbitmq-release-signing-key.asc | apt-key add -
apt-get update
apt-get install rabbitmq-server
````

#### 啟用 Web 管理及 MQTT/webMQTT 協定：
````
# RabbitMQ: tcp:5672, SSL:5671
rabbitmq-plugins enable rabbitmq_management       # tcp:15672, SSL:15671
rabbitmq-plugins enable rabbitmq_mqtt             # tcp:1883 , SSL:8883
rabbitmq-plugins enable rabbitmq_web_mqtt         # tcp:15675, SSL:15674
````

#### 下載 rabbitmqadmin：
````
cd /usr/local/bin
wget http://localhost:15672/cli/rabbitmqadmin
chmod +x rabbitmqadmin
````

#### 設定管理者帳號：
````
rabbitmqctl add_user admin admin8amma
rabbitmqctl set_user_tags admin administrator
rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
````
