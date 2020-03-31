---
title: Docker安装与使用
date: 2019-09-04 00:12:26
categories:
- 运维
tags:
- Docker
top:
---

[TOC]

### 一、Docker安装

参考资料：

docker官网：<https://docs.docker.com/install/linux/docker-ce/centos/#install-using-the-repository>

菜鸟教程：<https://www.runoob.com/docker/centos-docker-install.html>

安装环境：CentOS7



已经亲自操作可安装成功。

#### 1.移除旧版本

如果以前没安装过，就没必要执行：

```shell
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

#### 2.安装Docker Engine - Community

我们选择yum安装。手动和脚本安装参照官网。

##### 安装Docker repository

如果我们是第一次安装Docker，那么我们需要先安装Docker repository。这个东西用来安装和更新Docker。

1. 安装一些需要的包。`yum-utils` 提供 `yum-config-manager` 工具, 然后 `devicemapper` 的storage driver需要 `device-mapper-persistent-data` 和 `lvm2`。

   ```shell
   $ sudo yum install -y yum-utils \
     device-mapper-persistent-data \
     lvm2
   ```

2. 然后安装stable仓库。这里我们用阿里云镜像。[stable仓库](https://docs.docker.com/install/linux/docker-ce/centos/#install-using-the-repository)参考官网。

   ```shell
   $ sudo yum-config-manager \
       --add-repo \
       http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
   ```

##### 安装DOCKER ENGINE - COMMUNITY

1. 然后我们就可以安装引擎了。这里安装最新版。其他版本[参考官网](<https://docs.docker.com/install/linux/docker-ce/centos/#install-using-the-repository>)

```shell
$ sudo yum install docker-ce docker-ce-cli containerd.io
```

现在docker已经装好了，也生成了`docker`用户组。不过我们还没往里面加用户。

2. 启动Docker

``` shell
$ sudo systemctl start docker
```

3. 我们跑一个小Demo验证已经安装成功

```shell
$ sudo docker run hello-world
```

上面这条命令会下载一个测试的镜像，然后在容器里启用它。它执行一条命令后退出。

##### 其他

Docker 需要用户具有 sudo 权限，为了避免每次命令都输入`sudo`，可以把用户加入 Docker 用户组

```shell
$ sudo usermod -aG docker $USER
```

Docker 是服务器----客户端架构。命令行运行`docker`命令的时候，需要本机有 Docker 服务。如果这项服务没有启动，可以用下面的命令启动，二选一即可。

```shell
# service 命令的用法。service命令其实是去/etc/init.d目录下，去执行相关程序。
$ sudo service docker start

# systemctl 命令的用法。systemd对应的进程管理命令，兼容service。
$ sudo systemctl start docker
```

Docker不准确的说，可以看成是一个轻量的虚拟机。但其实更准确一点是把Docker可以看作一个shell端。