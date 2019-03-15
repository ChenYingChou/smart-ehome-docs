<h2>nginx & php7 建置</h2>

[[TOC]]

### 安裝 nginx
```sh
sudo su -
# 以下均以 root 權限執行

apt-get install -y nginx

# 擴充系統預設 I/O handles
cat > /etc/security/limits.d/99-nofile.conf << '_EOT_'
*               hard    nofile          51200
*               soft    nofile          51200
_EOT_

# 預建 OISP Web 目錄
mkdir -p /opt/oisp/www
ln -nfs opt/oisp /oisp
ln -nfs oisp/www /www
```

### 安裝 php7.2
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
apt-get install -t buster -y php7.2 php7.2-curl php7.2-gd php7.2-fpm php7.2-cli php7.2-opcache php7.2-mbstring php7.2-xml php7.2-zip php7.2-sqlite3 php7.2-dev
apt-get install -y php-pear
pecl channel-update pecl.php.net
sed -i 's/v_att_list = & func_get_args/v_att_list = func_get_args/' /usr/share/php/Archive/Tar.php
pecl install apcu
cat > /etc/php/7.2/mods-available/apcu.ini << '_EOT_'
; Enable APCu extension module
extension = apcu.so

;	This can be set to 0 to disable APCu
apc.enabled=1

;	Setting this enables APCu for the CLI version of PHP
;	(Mostly for testing and debugging).
apc.enable_cli=0

;	Sets the path to text files containing caches to load from disk upon
;	initialization of APCu. preload_path should be a directory where each
;	file follows $key.data where $key should be used as the entry name
;	and the contents of the file contains serialized data to use as the value
;	of the entry.
;apc.preload_path=

;	The size of each shared memory segment, with M/G suffixe
;apc.shm_size=32M

;	The number of seconds a cache entry is allowed to idle in a slot in case
;	this cache entry slot is needed by another entry.
;apc.ttl=0

;	The number of seconds that a cache entry may remain on the
;	garbage-collection list.
;apc.gc_ttl=3600

;	If you begin to get low on resources, an expunge of the cache
;	is performed if it is less than half full. This is not always
;	a suitable way of determining if an expunge of the cache
;	should be per apc.smart allows you to set a runtime configuration
;	value which	is used to determine if an expunge should be run
;	if (available_size < apc.smart * requested_size)
;apc.smart=0

;	A "hint" about the number variables expected in the cache.
;	Set to zero or omit if you are not sure;
;apc.entries_hint=4096

;	The mktemp-style file_mask to pass to the mmap module
apc.mmap_file_mask=/tmp/apc.XXXXXX

;	On very busy servers whenever you start the server or
;	modify files you can create a race of many processes
;	all trying to cache the same data at the same time.
;	By default, APCu attempts to prevent "slamming" of a key.
;	A key is considered "slammed" if it was the last key set,
;	and a context other than the current one set it ( ie. it
;	was set by another process or thread )
;apc.slam_defense=0

;	Defines which serializer should be used
;	Default is the standard PHP serializer.
;apc.serializer='php'

;	use the SAPI request start time for TTL
;apc.use_request_time=1

;	Enables APCu handling of signals, such as SIGSEGV, that write core files
;	when signaled. APCu will attempt to unmap the shared memory segment in
;	order to exclude it from the core file
;apc.coredump_unmap=0
_EOT_
ln -nfs /etc/php/7.2/mods-available/apcu.ini /etc/php/7.2/fpm/conf.d/40-apcu.ini
```

### nginx 設定
```sh
echo "NGINX Setup ..."
cd /etc/nginx

echo ">>> Create expires.conf"
cat > expires.conf << '_EOT_'
    location ~* \.(ico|css|js|gif|jpeg|jpg|png|woff|ttf|otf|svg|woff2|eot)$ {
        expires 30d;
        access_log off;
        add_header Pragma public;
        add_header Cache-Control "public";
    }
_EOT_

echo ">>> Patch fastcgi.conf"
patch -s -N -r - fastcgi.conf << '_EOT_'
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

echo ">>> Patch fastcgi_params"
patch -s -N -r - fastcgi_params << '_EOT_'
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

echo ">>> Create php.conf"
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
        fastcgi_param  PHP_ADMIN_VALUE  "open_basedir=/www:/oisp";
        include        fastcgi.conf;
    }
_EOT_

echo ">>> Patch nginx.conf"
patch -l -s -N -r - nginx.conf << '_EOT_'
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

echo ">>> Create sites-available/default"
cat > sites-available/default << '_EOT_'
# Default server configuration
server {
	listen 80 default_server;
	listen [::]:80 default_server;
	server_tokens off;

#-#	listen 443 ssl http2;
#-#	include ssl.conf;

	# disable content embedded in a frame or iframe
	add_header X-Frame-Options SAMEORIGIN; # DENY | SAMEORIGIN | ALLOW-FROM uri
	#add_header Access-Control-Allow-Origin *;
	#add_header Access-Control-Allow-Credentials true;

	root /oisp/www;
	index index.php index.html index.htm;
	server_name _;

	location / {
		# First attempt to serve request as file, then
		# as directory, then fall back to displaying a 404.
		try_files $uri $uri/ =404;
	}

	location /api {
		index		index.php;
		fastcgi_split_path_info ^(/api(?:/.+?\.php)?)($|/.*$);
		try_files	$uri $uri/ /api/index.php$fastcgi_path_info?$query_string;
	}

	location /auth {
		index		index.php;
		fastcgi_split_path_info ^(/auth(?:/.+?\.php)?)($|/.*$);
		try_files	$uri $uri/ /auth/index.php$fastcgi_path_info?$query_string;
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

# 重啟 nginx 服務
nginx -t && service nginx reload
```

### php 設定
```sh
echo "PHP Setup ..."
cd /etc/php/7.2/fpm

echo ">>> Patch php.ini"
patch -s -N -r - php.ini << '_EOT_'
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

echo ">>> Patch pool.d/www.conf"
patch -s -N -r - pool.d/www.conf << '_EOT_'
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

# 重啟 php-fpm 服務
service php7.2-fpm restart
```
