---
layout:     post
title:      "Docker基础"
subtitle:   "深入浅出Docker01"
date:       2022-07-02  16:46:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - MongoDB
    - DataBase
---

深入浅出Docker摘录第一部分，Docker基础

### 概览
- docker container run启动镜像为容器；docker container exec -it xxx bash连接到运行中的容器
- Dockerfile是纯文本文件，描述如何构建镜像；docker image build创建新镜像

### Docker引擎
- Docker主要由5部分组成：Client，daemon，containerd，shim和runc
- runc是OCI(开放容器计划)容器运行时规范的参考实现，负责创建容器
- containerd负责容器的生命周期管理，start/stop/pause/rm/...，它以daemon的方式运行
- daemon负责和Client交互，接收指令，并通知containerd完成具体工作；同时deamon还有镜像管理、身份认证、安全特性、核心网络和卷等功能
- 启动容器时，runc和操作系统通信，基于Namespace/CGroup等等功能创建容器，容器进程作为runc的子进程启动，启动完毕后runc将会退出，让关联的shim进程作为容器的父进程，和容器进行通信
- runc、shim和容器进程一一对应

### Docker镜像
- 镜像由一些松耦合的只读镜像层组成，多个镜像之间可以共享相同的镜像层
- 多架构镜像包含一个Manifest列表，每一项记录某种架构应该使用的具体镜像

### Docker容器
- 杀死容器中的主进程，容器也会被杀死
- 直至明确删除容器前，容器都不会丢失其中的数据；就算容器被删除了，如果数据保存在卷，数据也会保存下来
- docker container stop向进程发送SIGTERM，10S后发送SIGKILL，进程有10S清理并结束自己；而docker container rm会直接发送SIGKILL
- 启动容器(run)时可以用--restart指定重启策略，always除非手动stop，否则会自动重启，daemon重启后也会；unless-stopped类似，除了daemon重启后不会自动启动；on-failed在返回值不为0时重启，daemon重启也会自动启动

### 应用的容器化
- Dockerfile负责描述应用和容器化流程，第一行都是FROM，指定基础镜像层
- RUN会在基础镜像层的基础上，新建一个镜像层来保存执行结果内容；COPY复制指定文件或目录到镜像；两者都是新建镜像层
- WORKDIR指定后续指令的工作目录，EXPOSE设置对外端口，ENTRYPOINT指定镜像入口程序；三者都会增加元信息，不会新建镜像层
- 用&&连接多个命令可以减少镜像层数量，从而减少体积
- 镜像构建时如果发现某一层之前构建过，则可以使用这个缓存，但某一层没有命中缓存，后续都不能使用
