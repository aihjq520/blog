---
title: docker基础使用（二）
date: 2022/8/28 19:14:00   
tags: 
- docker
categories:         
- 运维
---


# Dockerfile 使用指南

本指南介绍了如何使用 Dockerfile 构建 Docker 镜像，我们将逐步介绍各个指令及其最佳实践。

## FROM

`FROM` 指令是定义一个基础镜像，是构建任何 Docker 镜像的起点。每个 Dockerfile 都必须以 `FROM` 开始。

## RUN

`RUN` 指令用于执行命令行命令，通常用于安装软件包和配置环境。`RUN` 具有两种格式：

**Shell 格式：**

例如

```dockerfile
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```

**Exec 格式：**

```
RUN ["可执行文件", "参数1", "参数2"]
```

例如：

```dockerfile
RUN ["echo", "<h1>Hello, Docker!</h1>", ">", "/usr/share/nginx/html/index.html"]
```

## 最佳实践：尽量合并多个命令

```dockerfile
FROM debian:jessie

RUN buildDeps='gcc libc6-dev make' \
    && apt-get update \
    && apt-get install -y $buildDeps \
    && wget -O redis.tar.gz "http://download.redis.io/releases/redis-3.2.5.tar.gz" \
    && mkdir -p /usr/src/redis \
    && tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
    && make -C /usr/src/redis \
    && make -C /usr/src/redis install \
    && rm -rf /var/lib/apt/lists/* \
    && rm redis.tar.gz \
    && rm -r /usr/src/redis \
    && apt-get purge -y --auto-remove $buildDeps
```

## 构建上下文

`COPY` 指令用于将文件从本地文件系统复制到镜像中。需要注意，复制的文件来自于上下文目录（context），而不是执行 docker build 命令所在的目录。

例如：

```dockerfile
COPY ./package.json /app/
```

这会复制上下文目录下的 package.json 到镜像中的 /app 目录。若路径超出上下文范围，将导致构建失败。因此，源文件路径必须是相对路径。

## COPY 和 ADD
`COPY` 和 ADD 指令都用于复制文件，但建议首选 COPY, 因其语义更明确。ADD 还具有解压功能，更适合用于需要自动解压的场景

## CMD

`CMD` 指令用于在容器启动时执行的命令，有两种格式：

### Shell 格式：

```dockerfile
CMD echo $HOME
```
实际的命令会被包装为 sh -c 的参数的形式进行执行。


```dockerfile
CMD ["sh", "-c", "echo $HOME"]
```

### Exec 格式：

```dockerfile
CMD ["echo", "$HOME"]
```

注意：容器中的应用应以前台方式运行，否则容器启动后会立即退出。例如：

```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

## ENTRYPOINT

`ENTRYPOINT` 指令与 `CMD` 类似，但更固定，且在运行时通常不被覆盖。当与 `CMD` 一起使用时，`CMD` 的内容作为参数传递给 `ENTRYPOINT`。

例如：
```dockerfile
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]
```

## ENV

`ENV` 指令用于设置环境变量，有两种格式：

```dockerfile
ENV <key> <value>
ENV <key1>=<value1> <key2>=<value2> ...
```

例如

```dockerfile
ENV APP_ENV=production
```

## VOLUME
`VOLUME` 指令用于声明匿名卷，使数据持久化并与容器存储层分离。适用于需要保存动态数据的应用（如数据库文件）。


### 概念

用于做容器内的数据持久化

## 挂载方式 Cli命令行方式

如果在启动容器的时候不指定-v，则就是匿名挂载

```shell

docker run -name mysql -p 3307:3306 -e MY_SQL_ROOT=123456 -d mysql:5.7 

```

以下是具名挂载，文件路径会和匿名放在一起，只是文件夹的名字换了

```shell

docker run -name mysql -p 3307:3306 -e MY_SQL_ROOT=123456 -v juming:/etc/lib/mysql -d mysql:5.7 

```

以下是指定路径挂载

```shell

docker run -name mysql -p 3307:3306 -e MY_SQL_ROOT=123456 -v /data/test:/etc/lib/mysql -d mysql:5.7 

```

### 挂载方式 dockerfile方式

```shell
FROM centos

VOLUME ['/volumn1', '/volumn2']
```


## EXPOSE
`EXPOSE` 指令声明容器运行时提供服务的端口。注意，这仅是声明，并不会自动进行端口映射。实际映射需要在运行时使用 -p 选项。

```dockerfile
docker run -p 8080:80 <image>
```