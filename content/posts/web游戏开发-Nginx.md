---
title: web游戏开发 nginx的部署
date: 2022-05-10T14:59:42+08:00
lastmod: 2022-05-10T14:59:42+08:00
# author: Author Name
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: /cover/lonely plannet.jpg
# images:
#   - /img/cover.jpg
categories:
  - web游戏开发
tags:
  - Django
  - nginx
  - uwsgi
# nolastmod: true
draft: false
---

web游戏开发日志，关于django的nginx的部署

<!--more-->

# 五、Nginx的部署

`django`的项目部署上可以采用`nginx`和`uwsgi`。如果采用nginx部署，则必须要要有域名，在阿里云上申请一个域名即可。

## 1、 关于uwsgi

`WSGI`协议是`Python`语言定义的 `Web` 服务器和 `Web` 应用程序或框架之间的一种简单而通用的接口。
所以简单来说`uWSGI`就是用来沟通`nginx`和`django`的一座桥梁。

`uWSGI`是一个`Web`服务器，它实现了`WSGI`协议、`uwsgi`、`http`等协议。`Nginx`中`HttpUwsgiModule`的作用是与`uWSGI`服务器进行交换。

## 2、关于nginx

`nginx`的作用总结有以下几点：

**安全**：程序不能直接被浏览器访问到，而是通过`nginx`,`nginx`只开放某个接口，`uwsgi`本身是内网接口，这样运维人员在`nginx`上加上安全性的限制，可以达到保护程序的作用。

**负载均衡**：一个`uwsgi`很可能不够用，即使开了多个`work`也是不行，毕竟一台机器的cpu和内存都是有限的，有了`nginx`做代理，一个`nginx`可以代理多台`uwsgi`完成`uwsgi`的负载均衡。

**静态文件处理高效**：用`django`或是`uwsgi`这种东西来负责静态文件的处理是很浪费的行为，而且他们本身对文件的处理也不如`nginx`好，所以整个静态文件的处理都直接由`nginx`完成，静态文件的访问完全不去经过`uwsgi`以及其后面的东西。

## 3、工作流程

`nginx` 是对外的服务接口，外部浏览器通过`url`访问`nginx`。`nginx` 接收到浏览器发送过来的`http`请求，将包进行解析，分析`url`，如果是静态文件请求就直接访问用户给`nginx`配置的静态文件目录，直接返回用户请求的静态文件，如果不是静态文件，而是一个动态的请求，那么`nginx`就将请求转发给`uwsgi`,`uwsgi` 接收到请求之后将包进行处理，处理成`wsgi`可以接受的格式，并发给`wsgi`,`wsgi`根据请求调用应用程序的某个文件，某个文件的某个函数，最后处理完将返回值再次交给`wsgi`,`wsgi`将返回值进行打包，打包成`uwsgi`能够接收的格式，`uwsgi`接收`wsgi` 发送的请求，并转发给`nginx`,`nginx`最终将返回值返回给浏览器。

## 4、具体操作

1）登录docker容器，关闭任务

2）返回服务器中，执行以下命令：

```
docker commit CONTAINER_NAME django:1.1  # 将容器保存成镜像，将CONTAINER_NAME替换成容器名称
docker stop CONTAINER_NAME  # 关闭容器
docker rm CONTAINER_NAME # 删除容器

# 使用保存的镜像重新创建容器
docker run -p 20000:22 -p 8000:8000 -p 80:80 -p 443:443 --name CONTAINER_NAME -itd django:1.1
```

3）要确保服务器**443端口**和**80端口**放行，即**https服务端口**和**http服务端口**。

4）配置nginx文件：

这一步很繁琐，自己操作的时候中间出了很多问题，尽量详细地记录一下流程和遇到的问题：

首先，要在域名管理的地方解析域名，我的域名是在阿里云上申请的，所以在阿里云域名管理的地方，解析域名，通过可以给网站添加前缀，然后映射到自己的服务器IP，然后在短暂的使用过后，应该就会提示你要备案了，不备案域名都访问不了，按照阿里云的流程一步一步走下去，正常个人一两天就能审核完成；

同时可以在还没被要求备案之前的一小段时间内，接着配置nginx。修改服务器`/etc/nginx/nginx.conf`文件，因为项目还需要使得整体动静分离，并且配置uwsgi，并且配置ssl，关于ssl的配置，可以直接在阿里云（其他服务器应该也有）直接申请免费的ssl服务，然后按照流程可以获得后缀为`.key`和`.pem`的文件，将`.key`中的内容写入服务器`/etc/nginx/cert/webapp.key`文件中。将`.pem`中的内容写入服务器`/etc/nginx/cert/webapp.pem`文件中（位置和名字是文件中配置的）。

nginx.conf文件在y总给的模板的基础上进行修改，并增加以下说明：

```
user root;   # 服务启动的用户名
worker_processes auto;  
pid /run/nginx.pid;  // 这个重复启动需要删除之前的这个文件
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 768;  # 设置进程的最大链接数
    # multi_accept on;  设置一个进程是否同时接受多个网络连接，默认为off
    # use epoll;      #事件驱动模型，select|poll|kqueue|epoll|resig|/dev/poll|eventport
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    include /etc/nginx/mime.types;  #文件扩展名与文件类型映射表
    default_type application/octet-stream;  #默认文件类型，默认为text/plain

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
    ssl_prefer_server_ciphers on;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;  # 这两个日志在开始之后注意权限问题，根据提示加权限

    gzip on;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;

     server {
         listen 80;  # 监听端口
         server_name bloodborne.shop;  # 域名
         location / {
               proxy_pass http://127.0.0.1:8000;  # 转发到的端口
           }
         # rewrite ^(.*)$ https://${server_name}$1 permanent;  # 把所有http改写为https
     }

    server {
        listen 443 ssl;
        server_name bloodborne.shop;
        ssl_certificate   cert/webapp.pem;
        ssl_certificate_key  cert/webapp.key;  # 这里要配置ssl证书，要把获取的文件填到该位置
        ssl_session_timeout 5m;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;
        charset utf-8;
        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;

        client_max_body_size 10M;

        location / {
            include /etc/nginx/uwsgi_params;
            uwsgi_pass 127.0.0.1:8000;  # 转发到uwsgi的8000端口
            uwsgi_read_timeout 60;

            if ($request_method = 'OPTIONS') {
                add_header 'Access-Control-Allow-Origin' 'https://www.bloodborne.shop';
                add_header 'Access-Control-Allow-Methods' 'GET, PUT, OPTIONS, POST, DELETE';
                add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization,X-Amz-Date';
                add_header 'Access-Control-Max-Age' 86400;
                add_header 'Content-Type' 'text/html; charset=utf-8';
                add_header 'Content-Length' 0;
                return 204;
            }
            if ($request_method = 'PUT') {
                add_header 'Access-Control-Allow-Methods' 'GET, PUT, OPTIONS, POST, DELETE';
                add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization,X-Amz-Date';
                add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';
            }
            if ($request_method = 'GET') {
                add_header 'Access-Control-Allow-Origin' 'https://www.bloodborne.shop';
                add_header 'Access-Control-Allow-Methods' 'GET, PUT, OPTIONS, POST, DELETE';
                add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization,X-Amz-Date';
                add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';
            }
        }
        location /static {  # 以下的静态文件，可以通过nginx转发直接访问
            alias /home/aria/webapp/static/;  # 配置静态文件地址
            if ($request_method = 'OPTIONS') {  # 下面添加跨域控制
                add_header 'Access-Control-Allow-Origin' 'https://www.bloodborne.shop';
                add_header 'Access-Control-Allow-Methods' 'GET, PUT, OPTIONS, POST, DELETE';
                add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization,X-Amz-Date';
                add_header 'Access-Control-Max-Age' 86400;
                add_header 'Content-Type' 'text/html; charset=utf-8';
                add_header 'Content-Length' 0;
                return 204;
            }
            if ($request_method = 'PUT') {
                add_header 'Access-Control-Allow-Methods' 'GET, PUT, OPTIONS, POST, DELETE';
                add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization,X-Amz-Date';
                add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';
            }
            if ($request_method = 'GET') {
                add_header 'Access-Control-Allow-Origin' 'https://www.bloodborne.shop';
                add_header 'Access-Control-Allow-Methods' 'GET, PUT, OPTIONS, POST, DELETE';
                add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization,X-Amz-Date';
                add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';
            }
        }
        location /media {
            alias /home/aria/webapp/media/;
            if ($request_method = 'OPTIONS') {
                add_header 'Access-Control-Allow-Origin' 'https://www.bloodborne.shop';
                add_header 'Access-Control-Allow-Methods' 'GET, PUT, OPTIONS, POST, DELETE';
                add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization,X-Amz-Date';
                add_header 'Access-Control-Max-Age' 86400;
                add_header 'Content-Type' 'text/html; charset=utf-8';
                add_header 'Content-Length' 0;
                return 204;
            }
            if ($request_method = 'PUT') {
                add_header 'Access-Control-Allow-Methods' 'GET, PUT, OPTIONS, POST, DELETE';
                add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization,X-Amz-Date';
                add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';
            }
            if ($request_method = 'GET') {
                add_header 'Access-Control-Allow-Origin' 'https://www.bloodborne.shop';
                add_header 'Access-Control-Allow-Methods' 'GET, PUT, OPTIONS, POST, DELETE';
                add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization,X-Amz-Date';
                add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range';
            }
        }
        location /wss {  # 配置wss信息
            proxy_pass http://127.0.0.1:5015;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header Host $http_host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }
}
```



5）启动nginx服务：

```
sudo /etc/init.d/nginx start
```

6) 修改django的配置

**打开`settings.py`文件**：
将域名添加到ALLOWED_HOSTS列表中。注意只需要添加https://后面的部分。令DEBUG = False。
**归档static文件**：
python3 manage.py collectstatic

7) 配置uwsgi

在django项目中添加uwsgi的配置文件：`scripts/uwsgi.ini`，内容如下：

```
[uwsgi]
socket          = 127.0.0.1:8000
chdir           = /home/aria/webapp
wsgi-file       = webapp/wsgi.py
master          = true
processes       = 2  # 进程数
threads         = 5  # 线程数
vacuum          = true
```

启动uwsgi服务：

```
uwsgi --ini scripts/uwsgi.ini
```

8）注意事项

如果之后修改了nginx的内容，需要`sudo nginx -s reload`重新加载一遍nginx文件更新内容。