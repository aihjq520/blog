---
title: docker基础使用（一）
date: 2022/7/27 11:28:00   
tags: 
- docker
categories: 
- 运维
---



### 架构

![](../images//docker.awebp)
左边大框框是我们进行 Docker 操作的宿主机，其运行了一个 Docker daemon 的核心守护程序，负责构建、运行和分发 Docker 容器。


在宿主机中安装了 Docker 客户端，其与 Docker daemon 守护进程进行通信，客户端会将 build、pull、run 等命令发送到  Docker daemon 守护进程进行执行。


右框框为 Docker 注册表存储 Docker 镜像，是一个所有 Docker 用户共享 Docker 镜像的服务，Docker daemon 守护进程与之进行交互。


下面是对架构中基本组成说明，比较详细，大家看的时候可以对着架构图看。概念这个东西，你看下就好，怎么记都记不住的，只有你常用的东西才会记住和想着去记住它，看完本文，可以把下面的应用实践一遍。

![](../images//docker2.awebp)

### Registry
镜像仓库，存储大量镜像，可以从镜像仓库拉取和推送镜像。

### Docker 镜像
类似虚拟机快照，从仓库拉取，或者在现有工具镜像上创建新镜像。通过镜像可以启动容器。
### Docker 容器
从镜像中创建应用环境，以单进程的方式运行。对外公开服务。是一种短暂的和一次性的环境。
### Docker 数据卷
数据卷可以完成数据持久化，数据卷是一个可供一个或多个容器使用的特殊目录，它绕过 UFS，可以提供很多有用的特性：

数据卷可以在容器之间共享和重用
对数据卷的修改会立马生效
对数据卷的更新，不会影响镜像
卷会一直存在，直到没有容器使用

### Docker 网络
Docker 容器之间的网络交互，可以使用端口映射的方式，其他容器可以直接通过端口实现。除该方式外还有一个容器连接（linking）系统也可以达到容器交互。（本文中 node 连接 mongodb 使用的是端口映射的方式）
关于Docker 网络模块，容器连接详情推荐这篇文章:
Docker的网络模式详解

### 镜像概念

镜像是一种轻量级，可执行的独立软件包，用来打包软件运行环境的和基于运行环境开发的软件。 它包含运行某个软件所需的所有内容，包括代码，运行时，库，环境变量和配置文件。

所有的应用，打包成docker镜像就可以直接跑起来。 

### 镜像加载原理

UnionFS(联合文件系统)： 是一种分层，轻量级并且高性能的文件系统，它支持对文件系统的修改作为一次提交来一层层的叠加。Union文件系统时Docker镜像的基础

bootfs, rootfs 名词概念
 

## 常用命令

```
1.镜像相关命令
1.1 搜索镜像
docker search 镜像名
1.2 下载镜像
docker pull 镜像名
1.3 查看本地镜像
docker images
1.4 删除镜像
docker rmi 镜像名
1.5 导出镜像
docker save -o xxx.tar 镜像名:版本
1.6 导入镜像
docker load < xxx.tar
2.容器相关命令
2.1 启动容器
docker run  -it -p 主机端口:容器端口 --name 容器名 镜像:TAG
#参数解释 run 运行容器 --name 指定容器的名称，但是不能和已经存在的重复 -d 后台运行容器，并返回容器ID -p 80:80 端口映射 宿主机端口:容器端口 -i 以交互模式运行容器，通常与 -t 同时使用； -t 为容器重新分配一个伪输入终端，通常与 -i 同时使用
2.2 停止容器
docker stop 容器名|容器ID
2.3 查看容器
docker ps     #查看正在运行的容器
docker ps -a  #查看所有容器
docker ps -q  #查看正在运行容器ID
docker ps -aq #查看所有容器ID
2.4 进入容器 
docker exec 会分配一个新的终端tty
docker exec -it 容器ID /bin/bash
#参数解释 exec 在运行的容器中执行命令 -i 保持终端打开 -t 分配一个终端
2.5 删除容器
docker rm 容器ID
批量删除容器
docker stop $(docker ps -q)
docker rm $(docker ps -aq)
2.6 启动已启动的容器
docker start 容器ID 启动一个已经停止的容器
docker restart 容器ID 重启容器 删除单个容器
2.7 从容器复制文件
docker cp CONTAINER:/容器路径 /宿主机路径
2.8 数据卷
宿主机目录：容器目录
docker run -it -v /home/ceshi:/home/ centos
```



### 私有镜像仓库

```SHELL
docker run -d -p 5000:5000 -v /opt/data/registry:/tmp/registry docker.io/registry
```


给基础镜像打个标签(前提是mysql镜像存在)

```SHELL
docker tag mysql 192.168.222.128:5000/mysql 
```


将镜像提交到私有仓库 
```SHELL
docker push 192.168.222.128:5000/mysql 
```



