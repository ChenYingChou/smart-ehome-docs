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
> 參考: [Raspberry Pi Dev Setup with Nginx + PHP7](https://getgrav.org/blog/raspberrypi-nginx-php7-dev)

```sh
echo "deb http://mirrordirector.raspbian.org/raspbian/ buster main contrib non-free rpi"  > /etc/apt/sources.list.d/10-buster.list

cat > /etc/apt/preferences.d/10-buster << '_EOT_'
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

#### 擴充系統預設 I/O handles
```sh
cat > /etc/security/limits.d/99-nofile.conf << '_EOT_'
*               hard    nofile          51200
*               soft    nofile          51200
_EOT_
```

#### nginx 設定
```sh
cd /etc/nginx
cat > expires << '_EOT_'
    location ~* \.(ico|css|js|gif|jpeg|jpg|png|woff|ttf|otf|svg|woff2|eot)$ {
        expires 30d;
        access_log off;
        add_header Pragma public;
        add_header Cache-Control "public";
    }
_EOT_

patch fastcgi.conf << '_EOT_'
--- fastcgi.conf~	2018-11-07 13:40:42.000000000 +0800
+++ fastcgi.conf	2017-12-02 03:30:21.130014475 +0800
@@ -6,6 +6,7 @@
 fastcgi_param  CONTENT_LENGTH     $content_length;

 fastcgi_param  SCRIPT_NAME        $fastcgi_script_name;
+fastcgi_param  PATH_INFO          $fastcgi_path_info;
 fastcgi_param  REQUEST_URI        $request_uri;
 fastcgi_param  DOCUMENT_URI       $document_uri;
 fastcgi_param  DOCUMENT_ROOT      $document_root;
_EOT_

patch fastcgi_params << '_EOT_'
--- fastcgi_params~	2018-11-07 13:40:42.000000000 +0800
+++ fastcgi_params	2017-12-02 03:37:46.678023454 +0800
@@ -5,6 +5,7 @@
 fastcgi_param  CONTENT_LENGTH     $content_length;

 fastcgi_param  SCRIPT_NAME        $fastcgi_script_name;
+fastcgi_param  PATH_INFO          $fastcgi_path_info;
 fastcgi_param  REQUEST_URI        $request_uri;
 fastcgi_param  DOCUMENT_URI       $document_uri;
 fastcgi_param  DOCUMENT_ROOT      $document_root;
_EOT_

cat > php.conf << '_EOT_'
    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    location ~ ^(.+?\.php)(/|$) {
       #try_files $fastcgi_script_name  =404; # --> clear $fastcgi_path_info
        if (!-f $document_root$fastcgi_script_name) { return 404; }
       #fastcgi_pass   127.0.0.1:9000;
        fastcgi_pass   unix:/run/php/php7.2-fpm.sock;
        fastcgi_index  index.php;
        fastcgi_split_path_info         ^(.+?\.php)($|/.*$);
        fastcgi_param  PHP_ADMIN_VALUE  "open_basedir=/www:/data/oisp";
        include        fastcgi.conf;
    }
_EOT_

patch -l nginx.conf << '_EOT_'
--- nginx.conf~	2018-11-07 13:40:42.000000000 +0800
+++ nginx.conf	2019-02-20 14:21:27.225859679 +0800
@@ -49,9 +49,11 @@
	gzip_disable "msie6";

	# gzip_vary on;
-	# gzip_proxied any;
+	gzip_min_length 1280;
+	gzip_proxied no-store no-cache private expired auth;
	# gzip_comp_level 6;
	# gzip_buffers 16 8k;
+	gzip_static on;
	# gzip_http_version 1.1;
	# gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
_EOT_

cat > conf.d/default.conf << '_EOT_'
# Default server configuration
server {
	listen 80 default_server;
	listen [::]:80 default_server;
	server_tokens off;

#-#	include ssl.conf;

	root /var/www/html;
	index index.php index.html index.htm;
	server_name _;

	location / {
		# First attempt to serve request as file, then
		# as directory, then fall back to displaying a 404.
		try_files $uri $uri/ =404;
	}

	# deny access to .htaccess files, if Apache's document root
	# concurs with nginx's one
	#
	location ~ /\.ht {
		deny all;
	}

	# Remove trailing slash to please routing system.
	if (!-d $request_filename) {
		rewrite	 ^/(.+)/$ /$1 permanent;
	}

	include php.conf;
	include expires.conf;
}
_EOT_
```

#### php 設定
```sh
cd /etc/php/7.2/fpm
patch php.ini << '_EOT_'
--- php.ini~	2019-02-15 20:59:13.488792831 +0800
+++ php.ini	2019-02-15 21:05:32.827256394 +0800
@@ -936,7 +936,7 @@
 [Date]
 ; Defines the default timezone used by the date functions
 ; http://php.net/date.timezone
-;date.timezone =
+date.timezone = "Asia/Taipei"

 ; http://php.net/date.default-latitude
 ;date.default_latitude = 31.7667
_EOT_

patch www.conf << '_EOT_'
--- www.conf~	2018-08-19 14:56:13.000000000 +0800
+++ www.conf	2019-02-15 21:17:13.324431309 +0800
@@ -337,7 +337,7 @@
 ; does not stop script execution for some reason. A value of '0' means 'off'.
 ; Available units: s(econds)(default), m(inutes), h(ours), or d(ays)
 ; Default Value: 0
-;request_terminate_timeout = 0
+request_terminate_timeout = 60s

 ; Set open file descriptor rlimit.
 ; Default Value: system defined value
@@ -420,4 +420,4 @@
 ;php_flag[display_errors] = off
 ;php_admin_value[error_log] = /var/log/fpm-php.www.log
 ;php_admin_flag[log_errors] = on
-;php_admin_value[memory_limit] = 32M
+php_admin_value[memory_limit] = 64M
_EOT_
```
