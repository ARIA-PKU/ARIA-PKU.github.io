---
title: Web游戏开发 准备工作
date: 2022-05-01T21:11:54+08:00
lastmod: 2022-05-01T21:11:54+08:00

cover: http://oss.surfaroundtheworld.top/blog-pictures/reverse_world.jpg
# images:
#   - /img/cover.jpg
categories:
  - web游戏开发
tags:
  - Django
  - jquery
# nolastmod: true
draft: false
---

web游戏开发日志

<!--more-->

# 一、项目准备

## 1、技术栈选择

实习期间，一直使用flask框架开发项目，这里选择django框架，熟悉python的另一个框架。项目总结自y总的django课程，项目后端选用django框架，前端使用jquery语言。服务器选用的是阿里云1核4G的学生机。

## 2、docker的配置

1）为了方便之后在不同的服务器中迁移自己的项目，所以项目整体选择在docker中搭建运行。在自己的服务器上下载好搭建了Django的镜像。

2）`docker load -i`将镜像从本地文件中加载进来

3）`docker run -p 8000:8000 -p 20000:22 --name django -itd django:1.0`创建并运行容器，其中`8000:8000`端口是开放`Http`,`20000:22`端口是开放`ssh`。（需要在服务器放行20000和8000端口）

## 3、git的配置

1）`ssh-keygen`生成密钥，并将其复制到github上，实现ssh的免密登录

2）`git init`  进到项目目录中将其配置成git仓库

3） 打开git，在git上创建一个仓库（项目）按照下面的提示在github里面配置一下git

`git config --global user.name xxx`

`git config --global user.email xxx@xxx.com`

`git add .`

`git commit -m "xxx"`

`git remote add origin ` 

 `git@xxx/XXX.git `  建立连接

`git push --set-upstream origin master`

4）在目录下创建`.gitignore`文件，规范上传文件的内容，比如`__pycache__`文件夹中的`.pyc`文件等，其中的`.pyc`文件是为了加速python运行而存在的，在项目中也可以用于单例模式实现。

## 4、项目的创建

1） `django-admin startproject webapp`，会在目录下创建一个名为`webapp`的文件夹，通过`tree .`命令查看结构如下：

```
`-- webapp
    |-- manage.py
    `-- webapp
        |-- __init__.py
        |-- asgi.py
        |-- settings.py
        |-- urls.py
        `-- wsgi.py
```

以上就是django的整体框架，将在以后的项目中应用各个模块。

2）`setting.py`即时整个项目默认的配置文件，在其中的`ALLOWED_HOSTS = []`中填入自己服务器的公网IP，和申请的域名地址，通过`python3 manage.py runserver 0.0.0.0:8000`命令，即可在自己服务器公网IP的8000端口看到最初的项目。

3）进入到`webapp`文件夹下，`python3 manage.py startapp game`生成自己的名为game的app，其树形结构如下：

```
`-- webapp
    |-- game
    |   |-- __init__.py
    |   |-- admin.py  # 管理员界面管理
    |   |-- apps.py  
    |   |-- migrations  # 数据库操作 
    |   |   `-- __init__.py
    |   |-- models.py  # 定义网站的数据类
    |   |-- tests.py  # 编写单元测试函数
    |   `-- views.py  # 编写视图函数
    |-- manage.py
    `-- webapp
        |-- __init__.py
        |-- __pycache__
        |   |-- __init__.cpython-38.pyc
        |   `-- settings.cpython-38.pyc
        |-- asgi.py
        |-- settings.py
        |-- urls.py  # 用于实现路由配置
        `-- wsgi.py

```

4）`python3 manage.py createsuperuser`创建超级管理员，并在`/admin`界面登录即可。

5） 在第二层的`webapp`文件夹下的`urls.py`为全局路由，可以通过配置实现各个函数的路由，为了便于扩展，在`game`文件夹下新建`url`文件夹存储之后的配置信息。