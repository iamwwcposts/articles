---
issueid: 36
tags:
- 翻译
title: Nginx如何处理请求？--翻译
date: 2020-03-13
updated: 2020-03-18
---
> 本文翻译自：
>
> http://nginx.org/en/docs/http/request_processing.html#simple_php_site_configuratio

### 基于域名的虚拟服务器

nginx首先确定使用哪个server来处理请求。让我们看下面简单的配置，这三个server都监听80端口

<!--more-->

```
server {
    listen 80;
    server_name example.org www.example.org;
...
}

server {
    listen 80;
    server_name example.net www.example.net;
...
}

server {
    listen 80;
    server_name example.com www.example.com;
...
}
```

在这个配置文件中，nginx只通过检测请求header的Host字段来决定哪个server处理请求。如果Host的值没有匹配到任何一个server_name，或者请求根本不包含Host字段，那么nginx会将请求路由到这个端口的默认server。在上面的配置文件中，nginx的default server是第一个server块。当然，默认server也可以明确被确定，只需要在listen指令中添加default_server参数。

```
server {
    listen 80 default_server;
    server_name example.net www.example.net
    ...
}
```

*default server参数从0.8.21版本开始提供。在早期的版本中，应该使用default参数*

记住，default server是监听端口的，而不是server_name的属性。后面还会提到。

### 如果丢弃掉没有Host字段的请求？

没有Host字段的请求是非法请求，server可以通过定义如下server来拒绝。

```
server {
    listen 80;
    server_name "";
    return 444;
}
```

空server_name将会被用来匹配没有Host字段的请求，我们直接返回nginx的444代码。

*从0.8.48版本开始，这已经是server name的默认设置的，所以 “” 可以省略不写。早期版本中默认server name将会使用操作系统的hostname*

### 混合使用基于域名和基于IP的虚拟主机

让我们看一个更复杂的配置。下面的虚拟主机监听的不同的端口。

```
server {
    listen 192.168.1.1:80;
    server_name example.org www.example.org;
...
}

server {
    listen 192.168.1.1:80;
    server_name example.net www.example.net;
...
}

server {
    listen 192.168.1.2:80;
    server_name example.com www.example.com;
...
}
```

在这个配置文件中，nginx首先检测请求的IP地址和端口并与server块中的listen指令对比。然后将请求的Host字段与server_name指令对照。如果找不到server_name，那么请求将会由默认server处理。比如，192.168.1.1:80端口收到的www.example.com的请求将会被192.168.1.1:80端口的默认server处理，这是因为www.example.com server_name没有在这个端口定义。

正如早已指出的，default_server是listen port的属性，我们可以在不同的端口定义不同的default server。

```
server {
    listen 192.168.1.1:80;
    server_name example.org www.example.org;
...
}

server {
    listen 192.168.1.1:80 default_server;
    server_name example.net www.example.net;
...
}

server {
    listen 192.168.1.2:80 default_server;
    server_name example.com www.example.com;
...
}
```

### 一个简单的PHP网站配置

现在让我们看下nginx如果选择location字段去处理一个典型的请求，如下是一个简易的PHP网站。

```
server {
    listen      80;
    server_name example.org www.example.org;
    root        /data/www;

    location / {
        index   index.html index.php;
    }

    location ~* \.(gif|jpg|png)$ {
        expires 30d;
    }

    location ~ \.php$ {
        fastcgi_pass  localhost:9000;
        fastcgi_param SCRIPT_FILENAME
                      $document_root$fastcgi_script_name;
        include       fastcgi_params;
    }
}
```

nginx首先搜索到最具体的location前缀而不会在意location出现的顺序。在上面的配置文件中，只有 `location /`被匹配，这是由于`/`可以匹配任何一个请求，但他会最后被使用。然后nginx按照个给出的顺序检测由正则表达式给出的`location`。第一个匹配的location会中断nginx的搜索，然后nginx会直接使用这个location，如果没有正则表达式匹配这个请求，那么nginx会使用最普适的location前缀，这里会使用刚才搜索到的`location /`

记住location字段仅仅检测URI部分，忽略query arguments。这样是由于query string会以多种不同的方式呈现。

```
/index.php?user=john&page=1
/index.php?page=1&user=john
```

除此之外，任何人都能够在query string里面添加任何信息

```
/index.php?page=1&something+else&user=john
```

现在让我们看下请求如何按照上面的配置文件处理

的

- 请求 `/logo.git`被前缀 / 首先匹配，然后被正则表达式 `\. (git|jpg|png)$`匹配。指令 `root /data/www`将 对于 `/logo.gif`映射成 `/data/www/logo.git`，于是文件发给了客户端

- 请求 `/index.php`也首先被 / 匹配，然后被 `\. (php)$`匹配。因此请求被之后的location处理，请求会被传递给监听在 `localhost:9000`的Fast CGI`。 fastcgi_param`指令将`SCRIPT_FILENAME`设置为 `/data/www/index.php`，然后 `FastCGI`服务器执行 `/data/www/index.php`文件。 `$document_root`相当于 `root`指令，变量 `$fastcgi_script_name`相当于请求的URI（/index.php） 。

- 请求 `about.html`仅仅被 / 匹配，因此它会被这个location处理，指令 `root /data/www`映射成了 `/data/www/about.html`，最后文件被发送给了客户端。

- 对于 / 的请求却更加复杂。它仅被 `/ location`匹配，因此它被这个location匹配。然后 `index`指令根据它的参数测试 index 文件是否存在。如果 `/data/www/index.html`文件不存在，但 `/data/www/index.php`文件存在，那么指令将会进行一次内部重定向，将请求定向到了 `/index.php`，nginx搜索再次搜索location块，这个请求就好像是由客户端发起的一样。正如我们之前看到了，被内部重定向的请求将会最终别 FastCGI服务处理。

译者注：

> 其实最后一种情况就是我们所说的伪静态，将 php 文件伪装成了 html 文件，内部重定向客户端察觉不到，可使用 `rewrite`指令完成伪静态。

