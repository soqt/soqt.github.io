---
layout: post
title: Python3 + pip3 + chrome + chromedriver 在centos上设置
categories: Python
description: Python config for my reference
keywords: keyword1, keyword2
date: 2018-11-25 00:00:00
hidden: true
---
爬虫学习笔记

<!-- more -->

准备工作
---------------
### python3
```
sudo apt-get update & sudo apt-get upgrade -y
sudo apt-get install build-essential libpcre3 libpcre3-dev zlib1g zlib1g-dev libssl-dev libgeoip-dev libgd-dev -y
```
> apt-get install apache2-utils 如果想用basic auth
### Chrome

安装chrome:
```
curl https://intoli.com/install-google-chrome.sh | bash

ldd /opt/google/chrome/chrome | grep "not found"
```

测试一下对百度拍一个snapshot：
```
google-chrome-stable --no-sandbox --headless --disable-gpu --screenshot https://www.baidu.com/
```

### Chromedriver
在Chromedriver的官网下载最新版本，移动到/opt/目录，不要移动到/opt/google目录，会报错，原因未知，然后link到/usr/local/bin中 `sudo ln -fs /opt/chromedriver /usr/local/bin/chromedriver`

在code里要关闭屏幕设置，代码如下
```
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

CHROMEDRIVER_PATH = '/opt/chromedriver'
WINDOW_SIZE = "1920,1080"

chrome_options = Options()
chrome_options.add_argument("--headless")
chrome_options.add_argument('--no-sandbox')
chrome_options.add_argument("--window-size=%s" % WINDOW_SIZE)

driver = webdriver.Chrome(executable_path=CHROMEDRIVER_PATH,
                        options=chrome_options
                        )

```

### 通用
```
mkdir ~/nginx_compile && cd ~/nginx_compile
wget -c https://nginx.org/download/nginx-1.15.5.tar.gz && 
tar -zxvf nginx-1.15.5.tar.gz && rm nginx-1.15.5.tar.gz **
rm nginx-1.15.5.tar.gz
```

安装依赖包
-----------------------------
### openSSL 1.1.1-pre2
```
cd ~/nginx_compile
wget -c https://www.openssl.org/source/openssl-1.1.1.tar.gz && 
tar zxf openssl-1.1.1.tar.gz && 
rm openssl-1.1.1.tar.gz
```

### ngx_brotli
Brotli是由Google的工程師所開發的一項壓縮演算法專案，目前運用在資料壓縮，當然主要是為了加快網頁的傳輸速度。目前Brotli已被各大主流瀏覽器支援，包含Chrome、Firefox、Edge與Safari等等。
```
git clone https://github.com/google/ngx_brotli.git
pushd ngx_brotli
git submodule update --init
popd
```

### Purge Cache

http://labs.frickle.com/nginx_ngx_cache_purge/
```
wget http://labs.frickle.com/files/ngx_cache_purge-2.3.tar.gz && tar -zxvf ngx_cache_purge-2.3.tar.gz && rm ngx_cache_purge-2.3.tar.gz
```

Config Nginx与安装
-----------------------------
```
 ./configure
 --sbin-path=/usr/bin/nginx
 --pid-path=/run/nginx.pid
 --conf-path=/etc/nginx/nginx.conf 
 --error-log-path=/var/log/nginx/error.log 
 --http-log-path=/var/log/nginx/access.log 
 --with-pcre 
 --with-http_image_filter_module=dynamic 
 --modules-path=/etc/nginx/modules 
 --with-http_v2_module 
 --with-http_ssl_module 
 --with-http_gzip_static_module 
 --without-http_autoindex_module
 --with-http_geoip_module
 --with-openssl=../openssl-1.1.1
 --add-module=../ngx_brotli
 --add-module=../ngx_cache_purge-2.3
 --with-http_realip_module
```
--sbin-path nginx安装位置  
--conf-path config文件位置  
--with-pcre 用pcre library(regex)  
--pid-path=/var/run/nginx.pid  pid位置  
``` 
sudo make 
sudo make install  
sudo nginx
``` 
 
Systemd Settings:
-----------------------------
```
sudo nginx -s stop
```
go to https://www.nginx.com/resources/wiki/start/topics/examples/systemd/

create systemd file and copy paste. Remember to modify the path to the correct path we configured.
```
sudo touch /lib/systemd/system/nginx.service
sudo vim /lib/systemd/system/nginx.service
```
```
[Unit]
Description=The NGINX HTTP and reverse proxy server
After=syslog.target network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/bin/nginx -t
ExecStart=/usr/bin/nginx
ExecReload=/usr/bin/nginx -s reload
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
```
sudo systemctl start nginx
sudo systemctl enable nginx    //enable nginx restart when reboot
```

Certbot  (尽量别用，因为会修改nginx的configure，腾讯云有一年免费SSL证书)
----------------------------------------------------------
去官网安装certbot
修改 nginx.conf 中 http > server > server_name example.com
**安装过程中全部enter跳过**

```
# Server
location ^~ /.well-known/acme-challenge/ {
   default_type "text/plain";
   root     /usr/share/nginx/html;
}

location = /.well-known/acme-challenge/ {
   return 404;
}
```

sudo service nginx reload
```
sudo certbot certonly --webroot -w /usr/share/nginx/html/ -d your.domain.com
```
renew:
```
sudo crontab -e
```
```
@daily sudo certbot renew
```

添加 Brotli
-----------------------------
```
gzip on;
    # ... （Gzip壓縮格式的其他設定。）
## brotli Compression.
brotli on;
brotli_comp_level 6;
#『brotli_types』的值僅作參考，請依你的環境去做設定。
brotli_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript image/x-icon application/vnd.ms-fontobject font/opentype application/x-font-ttf;
```

安全
----------------------------
安装Diffie-Hellman 
```
sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```