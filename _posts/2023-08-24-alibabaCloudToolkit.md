---
layout: post
title: alibaba cloud toolkit的使用和部署
categories: 服务器
description: 服务器部署的便捷方式
keywords: 部署
---

## 问题

以往的写好的应用程序放到服务器上部署的方式都是在本地打包成jar包，传到服务器上，在服务器用命令行关闭原版本的应用程序，在启动新版本的应用程序，每次写好一个功能要与前端联调都要经历这些繁琐的步骤，在使用alibaba cloud toolkit这款IDEA插件后，则无需这些繁琐步骤，节省时间提高效率。

## 使用教程

在IDEA插件market里面搜索alibaba cloud toolkit，并下载和安装。

安装后选择add host

![image-20230824194657540](https://raw.githubusercontent.com/scottyzh/scottyzh.github.io/main/image/image-20230824194657540.png)

在里面添加服务器ip和账号密码

![image-20230824192733428](https://raw.githubusercontent.com/scottyzh/scottyzh.github.io/main/image/image-20230824192733428.png)

在下面选择jar包，并且jar包的上传目录，以及上传后执行的命令，这里面我写了个sh文件，jar包是采用docker部署，在sh文件里面写了相关的命令行

![image-20230824193403662](https://raw.githubusercontent.com/scottyzh/scottyzh.github.io/main/image/image-20230824193403662.png)

在jar包的目录放了dockerfile与相关的.sh执行文件

dockerfile的内容：

```
# 指定基础镜像，来构建此镜像，可以理解为运行的需要基础环境
FROM openjdk:8
# 维护者信息
MAINTAINER Reed
# 定义匿名卷
VOLUME /tmp
# 创建挂载目录
RUN mkdir -p /preview_workdir/sysRes
#WORKDIR指令用于指定容器的一个目录， 容器启动时执行的命令会在该目录下执行。
WORKDIR /preview_workdir
##将当前jar包复制到容器对应目录下并修改名称
ADD Plan-1.0-SNAPSHOT.jar /preview_workdir/app.jar
# 允许指定的端口
EXPOSE 8183
# 入口
ENTRYPOINT ["java","-jar","/preview_workdir/app.jar"]

```

sh文件的内容：

```
#!/bin/bash

cd /home/preview/

docker rm -f preview

docker rmi -f preview:v2.0

docker build -t preview:v2.0 ./

docker run -p 8183:8183 -v /data/docker/preview_v/sysRes:/preview_workdir/sysRes --name preview --restart unless-stopped -d preview:v2.0

docker logs --tail 1000 preview
```

之后点击upload按钮，即可实现jar的上传和启动
