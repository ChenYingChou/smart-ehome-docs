nginx & php7 建置
---

#### 以 root 權限執行安裝 nginx：
```sh
apt-get update
apt-get upgrade
apt-get remove apache2
apt-get install nginx
systemctl start nginx
```

#### 以 root 權限執行安裝 php7.2
> 註: php7.2 或 php7.0 請選其中之一安裝即可 \
> 參考: [Raspberry Pi Dev Setup with Nginx + PHP7](https://getgrav.org/blog/raspberrypi-nginx-php7-dev)

```sh
echo "deb http://mirrordirector.raspbian.org/raspbian/ buster main contrib non-free rpi"  > /etc/apt/sources.list.d/10-buster.list

cat > /etc/apt/preferences.d/10-buster <<< _EOT_
Package: *
Pin: release n=stretch
Pin-Priority: 900

Package: *
Pin: release n=buster
Pin-Priority: 750
_EOT_
apt-get update
apt-get install -t buster php7.2 php7.2-curl php7.2-gd php7.2-fpm php7.2-cli php7.2-opcache php7.2-mbstring php7.2-xml php7.2-zip
apt-get install php-pear
apt-get install php7.2-dev
pecl channel-update pecl.php.net
pecl install apcu
```

#### 以 root 權限執行安裝 php7.0：
> 註: php7.2 或 php7.0 請選其中之一安裝即可 \
> 參考: [Raspberry Pi NGINX: Build your own Web Server](https://pimylifeup.com/raspberry-pi-nginx/)

```sh
echo "deb http://mirrordirector.raspbian.org/raspbian/ stretch main contrib non-free rpi" >> /etc/apt/sources.list
cat >> /etc/apt/preferences <<< _EOT_

Package: *
Pin: release n=jessie
Pin-Priority: 600
_EOT_
apt-get update
apt-get install -t stretch php7.0-fpm
```

#### 擴充系統預設 I/O handles
```sh
cat > /etc/security/limits.d/99-nofile.conf << _EOT_
*               hard    nofile          51200
*               soft    nofile          51200
_EOT_
```

#### nginx 設定
```sh
cat > /etc/nginx/expires << _EOT_
    location ~* \.(ico|css|js|gif|jpeg|jpg|png|woff|ttf|otf|svg|woff2|eot)$ {
        expires 30d;
        access_log off;
        add_header Pragma public;
        add_header Cache-Control "public";
    }
_EOT_

cat > /etc/nginx/fastcgi.conf << _EOT_
fastcgi_param  SCRIPT_FILENAME    $document_root$fastcgi_script_name;
fastcgi_param  QUERY_STRING       $query_string;
fastcgi_param  REQUEST_METHOD     $request_method;
fastcgi_param  CONTENT_TYPE       $content_type;
fastcgi_param  CONTENT_LENGTH     $content_length;

fastcgi_param  SCRIPT_NAME        $fastcgi_script_name;
fastcgi_param  PATH_INFO          $fastcgi_path_info;
fastcgi_param  REQUEST_URI        $request_uri;
fastcgi_param  DOCUMENT_URI       $document_uri;
fastcgi_param  DOCUMENT_ROOT      $document_root;
fastcgi_param  SERVER_PROTOCOL    $server_protocol;
fastcgi_param  REQUEST_SCHEME     $scheme;
fastcgi_param  HTTPS              $https if_not_empty;

fastcgi_param  GATEWAY_INTERFACE  CGI/1.1;
fastcgi_param  SERVER_SOFTWARE    nginx/$nginx_version;

fastcgi_param  REMOTE_ADDR        $remote_addr;
fastcgi_param  REMOTE_PORT        $remote_port;
fastcgi_param  SERVER_ADDR        $server_addr;
fastcgi_param  SERVER_PORT        $server_port;
fastcgi_param  SERVER_NAME        $server_name;

# PHP only, required if PHP was built with --enable-force-cgi-redirect
fastcgi_param  REDIRECT_STATUS    200;
_EOT_

cat > /etc/nginx/php.conf << _EOT_
    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    location ~ ^(.+?\.php)(/|$) {
       #try_files $fastcgi_script_name  =404; # --> clear $fastcgi_path_info
        if (!-f $document_root$fastcgi_script_name) { return 404; }
       #fastcgi_pass   127.0.0.1:9000;
        fastcgi_pass   unix:/run/php-fpm/php-fpm.sock;
        fastcgi_index  index.php;
        fastcgi_split_path_info         ^(.+?\.php)($|/.*$);
        fastcgi_param  PHP_ADMIN_VALUE  "open_basedir=/www:/data/mq-data";
        include        fastcgi.conf;
    }
_EOT_

cat > /etc/nginx/nginx.conf << _EOT_
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# setsebool -P httpd_setrlimit=1
worker_rlimit_nofile 51200;

# Load dynamic modules. See /usr/share/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 16384;
    multi_accept on;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    log_format scripts '$document_root$fastcgi_script_name$fastcgi_path_info ?$query_string > $request';

    access_log  /var/log/nginx/access.log  main;

    client_header_timeout     12; # 1m
    client_body_timeout       12; # 1m
    send_timeout              10; # 1m
    keepalive_timeout         15;
    reset_timedout_connection on;

    open_file_cache           max=1024 inactive=60s;
    open_file_cache_valid     30s;
    open_file_cache_min_uses  5;
    open_file_cache_errors    off;

    sendfile                  on;
    tcp_nopush                on;
    tcp_nodelay               on;
    types_hash_max_size       2048;

    gzip                      on;
    gzip_vary                 on;
    gzip_min_length           1280;
    gzip_types                text/plain text/css application/json application/javascript application/x-javascript text/xml application/xml application/xml+rss text/javascript;
    gzip_proxied              no-store no-cache private expired auth;
    gzip_static               on;
    gzip_disable              "MSIE [1-6]\.(?!.*SV1)";
    gzip_http_version         1.0;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;
}
_EOT_

cat > /etc/nginx/conf.d/default.conf << _EOT_
#
# The default server
#
geo $limited {
    default             1;
    127.0.0.0/16        0;
    192.168.0.0/16      0;
    59.124.5.205/32     0;
    59.124.5.206/32     0;
    59.124.5.207/32     0;
    114.35.97.8/32      0;      # finenet.com.tw
}

map $limited $limit {
    1   $binary_remote_addr;
    0   "";
}

limit_req_zone $limit zone=limited:1m rate=3r/s;
limit_req zone=limited burst=30;

limit_conn_zone $binary_remote_addr zone=perip:2m;
#limit_zone perip $binary_remote_addr 1m;

server {
    listen 127.0.0.1:8001 default_server;

    location /auth {
        root        /data/mq-data/www;
        index       index.php;
        fastcgi_split_path_info ^(/auth(?:/.+?\.php)?)($|/.*$);
        try_files   $uri $uri/ /auth/index.php$fastcgi_path_info?$query_string;
        include     php.conf;
    }

    error_page  404              /404.html;
    location = /404.html {
        root   /usr/share/nginx/html;
    }
}

server {
    listen      80 default_server;
    server_name oisp.smart-ehome.com oisp;
    server_tokens off;

   #include ssl.conf;

    # disable content embedded in a frame or iframe
    add_header X-Frame-Options SAMEORIGIN; # DENY | SAMEORIGIN | ALLOW-FROM uri
   #add_header Access-Control-Allow-Origin *;
   #add_header Access-Control-Allow-Credentials true;

    # debuging for scripts
   #access_log  /var/log/nginx/scripts.log scripts;
   #access_log  /var/log/nginx/host.access.log  main;

    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;

    root /www;
    location / {
        #root   /usr/share/nginx/html;
        index  index.php index.html index.htm;
       #auth_basic "OISP";
       #auth_basic_user_file /home/www/htuserA;
        include php.conf;
        include expires.conf;
    }

    location /public {
        index  index.php index.html index.htm;
    }

    location = /nginx_status {
        # Turn on stats
        stub_status on;
        access_log  off;
        allow 127.0.0.1;
        allow 192.168.0.0/16;
        allow 114.35.97.8;
        deny all;
    }

    location /icon/ {
        alias   /usr/share/nginx/html/;
    }

    error_page  404              /404.html;
    location = /404.html {
        root   /usr/share/nginx/html;
    }

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    location ~ /\.ht {
        deny  all;
    }

    location /.well-known/acme-challenge/ {
        alias /home/www/challenge/;
        try_files $uri =404;
    }

    # Remove trailing slash to please routing system.
    if (!-d $request_filename) {
        rewrite     ^/(.+)/$ /$1 permanent;
    }

    include php.conf;
    include expires.conf;

    limit_conn perip 6;
    limit_rate_after 2m;
    if ($limited) {
        #set $limit_rate 250k;  # replace with "limit_rate_after 1m" / "set $limit_rate 200k"
        set $limit_rate 500k;
    }
}
_EOT_
```

#### php 設定
```sh
cd /etc
yum install -y patch
patch php.ini << _EOT_
--- php.ini	2017-02-18 22:49:58.000000000 +0800
+++ /tmp/php.ini	2018-02-02 13:24:33.564607146 +0800
@@ -302,7 +302,7 @@
 ; It receives a comma-delimited list of function names. This directive is
 ; *NOT* affected by whether Safe Mode is turned On or Off.
 ; http://php.net/disable-functions
-disable_functions =
+disable_functions = show_source

 ; This directive allows you to disable certain classes for security reasons.
 ; It receives a comma-delimited list of class names. This directive is
@@ -363,7 +363,7 @@
 ; threat in any way, but it makes it possible to determine whether you use PHP
 ; on your server or not.
 ; http://php.net/expose-php
-expose_php = On
+expose_php = Off

 ;;;;;;;;;;;;;;;;;;;
 ; Resource Limits ;
@@ -779,7 +779,7 @@

 ; Whether to allow HTTP file uploads.
 ; http://php.net/file-uploads
-file_uploads = On
+file_uploads = Off

 ; Temporary directory for HTTP uploaded files (will use system default if not
 ; specified).
@@ -799,7 +799,7 @@

 ; Whether to allow the treatment of URLs (like http:// or ftp://) as files.
 ; http://php.net/allow-url-fopen
-allow_url_fopen = On
+allow_url_fopen = Off

 ; Whether to allow include/require to open URLs (like http:// or ftp://) as files.
 ; http://php.net/allow-url-include
@@ -866,7 +866,7 @@
 [Date]
 ; Defines the default timezone used by the date functions
 ; http://php.net/date.timezone
-;date.timezone =
+date.timezone = "Asia/Taipei"

 ; http://php.net/date.default-latitude
 ;date.default_latitude = 31.7667
_EOT_

patch www.conf << _EOT_
--- www.conf.orig	2018-02-02 18:25:51.970702632 +0800
+++ www.conf	2018-02-02 18:28:00.118139065 +0800
@@ -23,7 +23,7 @@
 ; RPM: apache Choosed to be able to access some dir as httpd
 user = apache
 ; RPM: Keep a group allowed to write in log dir.
-group = apache
+group = nginx

 ; The address on which to accept FastCGI requests.
 ; Valid syntaxes are:
@@ -35,7 +35,8 @@
 ;                            (IPv6 and IPv4-mapped) on a specific port;
 ;   '/path/to/unix/socket' - to listen on a unix socket.
 ; Note: This value is mandatory.
-listen = 127.0.0.1:9000
+;listen = 127.0.0.1:9000
+listen = /run/php-fpm/php-fpm.sock

 ; Set listen(2) backlog.
 ; Default Value: 511
@@ -45,8 +46,8 @@
 ; permissions must be set in order to allow connections from a web server.
 ; Default Values: user and group are set as the running user
 ;                 mode is set to 0660
-;listen.owner = nobody
-;listen.group = nobody
+listen.owner = apache
+listen.group = nginx
 ;listen.mode = 0660

 ; When POSIX Access Control Lists are supported you can set them using
@@ -106,22 +107,22 @@
 ; forget to tweak pm.* to fit your needs.
 ; Note: Used when pm is set to 'static', 'dynamic' or 'ondemand'
 ; Note: This value is mandatory.
-pm.max_children = 50
+pm.max_children = 30

 ; The number of child processes created on startup.
 ; Note: Used only when pm is set to 'dynamic'
 ; Default Value: min_spare_servers + (max_spare_servers - min_spare_servers) / 2
-pm.start_servers = 5
+pm.start_servers = 3

 ; The desired minimum number of idle server processes.
 ; Note: Used only when pm is set to 'dynamic'
 ; Note: Mandatory when pm is set to 'dynamic'
-pm.min_spare_servers = 5
+pm.min_spare_servers = 3

 ; The desired maximum number of idle server processes.
 ; Note: Used only when pm is set to 'dynamic'
 ; Note: Mandatory when pm is set to 'dynamic'
-pm.max_spare_servers = 35
+pm.max_spare_servers = 10

 ; The number of seconds after which an idle process will be killed.
 ; Note: Used only when pm is set to 'ondemand'
@@ -132,7 +133,7 @@
 ; This can be useful to work around memory leaks in 3rd party libraries. For
 ; endless request processing specify '0'. Equivalent to PHP_FCGI_MAX_REQUESTS.
 ; Default Value: 0
-;pm.max_requests = 500
+pm.max_requests = 5000

 ; The URI to view the FPM status page. If this value is not set, no URI will be
 ; recognized as a status page. It shows the following informations:
@@ -329,7 +330,7 @@
 ; does not stop script execution for some reason. A value of '0' means 'off'.
 ; Available units: s(econds)(default), m(inutes), h(ours), or d(ays)
 ; Default Value: 0
-;request_terminate_timeout = 0
+request_terminate_timeout = 60s

 ; Set open file descriptor rlimit.
 ; Default Value: system defined value
@@ -412,7 +413,7 @@
 ;php_flag[display_errors] = off
 php_admin_value[error_log] = /var/log/php-fpm/www-error.log
 php_admin_flag[log_errors] = on
-;php_admin_value[memory_limit] = 128M
+php_admin_value[memory_limit] = 64M

 ; Set the following data paths to directories owned by the FPM process user.
 ;
_EOT_
```
