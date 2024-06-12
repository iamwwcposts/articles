---
issueid: 35
tags:
title: Nginx配置Oneindex
date: 2020-03-12
updated: 2020-03-12
---
每一个 `server` 都是一个虚拟主机，通过 `http` 的 `host` 字段区分不同的目录

这个 `host` 字段对应着`nginx`的`server_name`

`oneindex`需要 `php-fpm` 来处理请求，`php-fpm` 默认使用`unix socket`，需要在 `/etc/php/7.0/fpm/pool.d/`下的配置文件中添加

```
listen = 127.0.0.1:9000
listen = /run/php/php7.0-fpm.sock
```
使得 `php-fpm` 监听 `9000` 端口

nginx根目录为 `/var/www/html`，我将 `oneindex` 放在了 `/var/www/html/oneindex`

如下为配置文件

```
server {
  listen  80;
  # 指定我使用的域名
  server_name drive.chaochaogege.com;
  index index.php;
 # 虚拟主机的根目录
  root /var/www/html/oneindex;
  location / {
      index index.html;
      #Implementing PHP pseudo static
      try_files $uri /index.php?$args;
  }
  location ~ \.php$ {
    fastcgi_pass  localhost:9000;
    fastcgi_index index.php;
    include fastcgi_params;
   #document_root 与上面的root相当，指的是请求的根目录
    fastcgi_param SCRIPT_FILENAME $document_root/$fastcgi_script_name;
    # fastcgi_param QUERY_STRING    $query_string;
  }
}
```