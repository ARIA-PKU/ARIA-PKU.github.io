---
title: Vscode连接服务器
date: 2022-05-04T15:34:40+08:00
lastmod: 2022-05-04T15:34:40+08:00
# author: Author Name
# avatar: /img/author.jpg
# authorlink: https://author.site
cover: /img/cover.jpg
# images:
#   - /img/cover.jpg
categories:
  - IDE使用方法
tags:
  - vscode
# nolastmod: true
draft: false
---

关于如何使用vscode连接服务器并配置免密登录以及遇到的问题记录。

<!--more-->

## vscode中安装remote-ssh

因为vscode的本质就是利用的ssh来建立连接的；

## 配置文件
创建文件 `~/.ssh/config`。

然后在文件中输入：

```
Host myserver1
    HostName IP地址或域名
    User 用户名

Host myserver2
    HostName IP地址或域名
    User 用户名
之后再使用服务器时，可以直接使用别名myserver1、myserver2。
```

## 密钥登录
创建密钥：

`ssh-keygen`
然后一直回车即可。

执行结束后，~/.ssh/目录下会多两个文件：

```
id_rsa：私钥
id_rsa.pub：公钥
```


之后想免密码登录哪个服务器，就将公钥传给哪个服务器即可。

例如，想免密登录myserver服务器。则将公钥中的内容，复制到myserver中的~/.ssh/authorized_keys文件里即可。

也可以使用如下命令一键添加公钥：

`ssh-copy-id myserver`

## docker下配置ssh的问题记录

首先，为了方便迁移项目采用docker配置项目环境，通过服务器的20000端口映射到docker内的22端口，以实现远程连接；

通过以上操作是可以正常连接的，但是在我给项目设置完nginx之后，发现vscode连接不上服务了，然后根据网上的方法开始排查：

1、端口开放，早就开放了20000端口了，所以无论在阿里云的安全组里还是服务器`nestat -tunlp|grep 20000`都能找到20000对应的端口是正常的，本地telnet这个端口无法连接成功；

2、有的说是可能是IP被服务器拉黑了，但是搜了一下，阿里云好像是没有自动进行黑名单IP设置的，只有白名单功能，添加了之后也没有解决问题，换了一台服务器，发现也连接不上开发服务器的20000端口，可以确定不是IP的问题了；

3、还有的说是半连接的太多了，导致了连接失败，ssh是通过tcp连接的，tcp连接过程的半连接问题，但是这个通过`netstat`命令查看，发现也不是这个原因；

4、最后是说ssh服务没有开启，我在服务器`ps -aux|grep ssh`发现ssh服务是正常运行的，并且telnet测试服务器22端口也是正常的，然后就开始把服务器各种服务重启试了一遍，发现都没有效果。休息的时候，我才突然想到，在连接20000端口用的ssh其实是docker里面的ssh服务，而我排查的是服务器的ssh服务........

5、在docker内部`ps -aux|grep ssh`，终于发现，ssh在docker内没有正常启动，然后启动ssh的时候，发现提示了下面的问题：

`/run/sshd must be owned by root and not group or world-writable`，应该是之前设置nginx的时候，给文件开权限出的问题，

网上搜了一下解决办法：

```
chown -R root.root /run/sshd
chmod 744 /run/sshd
service sshd restart
```

折腾了一晚上，终于又能正常连接vscode了，以后还是不要随便给文件开权限了.....

