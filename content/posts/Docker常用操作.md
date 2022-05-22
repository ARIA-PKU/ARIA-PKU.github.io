---
title: Docker常用操作
date: 2022-04-30T20:32:39+08:00
lastmod: 2022-04-30T20:32:39+08:00
author: ARIA
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: /cover/docker.jpg
# images:
#   - /img/cover.jpg
categories:
  - Docker
tags:
  - Docker常用操作
# nolastmod: true
draft: false
---

Docker常用命令整理

<!--more-->

# Docker 常用操作

## 一、将当前用户添加到docker用户组

为了避免每次使用docker命令都需要加上sudo权限，可以将当前用户加入安装中自动创建的docker用户组(可以参考官方文档（https://docs.docker.com/engine/install/linux-postinstall/）)：

sudo usermod -aG docker $USER
执行完此操作后，需要退出服务器，再重新登录回来，才可以省去sudo权限。

## 二、镜像（images）

1. docker pull ubuntu:20.04：拉取一个镜像
2. docker images：列出本地所有镜像
3. docker image rm ubuntu:20.04 或 docker rmi ubuntu:20.04：删除镜像ubuntu:20.04
4. docker [container] commit CONTAINER IMAGE_NAME:TAG：创建某个container的镜像
5. docker save -o ubuntu_20_04.tar ubuntu:20.04：将镜像ubuntu:20.04导出到本地文件ubuntu_20_04.tar中
6. docker load -i ubuntu_20_04.tar：将镜像ubuntu:20.04从本地文件ubuntu_20_04.tar中加载出来

## 三、容器(container)

1. docker [container] create -it ubuntu:20.04：利用镜像ubuntu:20.04创建一个容器。

2. docker ps -a：查看本地的所有容器

3. docker [container] start CONTAINER：启动容器

4. docker [container] stop CONTAINER：停止容器

5. docker [container] restart CONTAINER：重启容器

6. docker [contaienr] run -itd ubuntu:20.04：创建并启动一个容器

    e.g. docker run -p 20000:22 -p 8000:8000 --name django_project  -itd django_lesson 其中-p为端口映射，django_lesson:1.0为镜像，django_project为项目名称

7. docker [container] attach CONTAINER：进入容器
   先按Ctrl-p，再按Ctrl-q可以挂起容器（这里用Ctrl-d是直接退出并关闭容器）

8. docker [container] exec CONTAINER COMMAND：在容器中执行命令

9. docker [container] rm CONTAINER：删除容器

10. docker container prune：删除所有已停止的容器

11. docker export -o xxx.tar CONTAINER：将容器CONTAINER导出到本地文件xxx.tar中

12. docker import xxx.tar image_name:tag：将本地文件xxx.tar导入成镜像，并将镜像命名为image_name:tag

13. docker export/import与docker save/load的区别： export/import会丢弃历史记录和元数据信息，仅保存容器当时的快照状save/load会保存完整记录，体积更大

14. docker top CONTAINER：查看某个容器内的所有进程

15. docker stats：查看所有容器的统计信息，包括CPU、内存、存储、网络等信息

16. docker cp xxx CONTAINER:xxx 或 docker cp CONTAINER:xxx xxx：在本地和容器间复制文件

17. docker rename CONTAINER1 CONTAINER2：重命名容器

18. docker update CONTAINER --memory 500MB：修改容器限制

## 四、注意事项

考虑到权限问题容易引发的问题，和linux用户创建一样，创建一个root以外的用户，并赋给其sudo权限：

adduser aria # 创建普通用户aria

usermod -aG sudo aria # 给用户aria分配sudo权限

su -aria# 可切换到用户aria中