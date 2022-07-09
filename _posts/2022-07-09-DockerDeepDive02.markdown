---
layout:     post
title:      "Docker进阶"
subtitle:   "深入浅出Docker02"
date:       2022-07-09  20:46:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - MongoDB
    - DataBase
---

深入浅出Docker摘录第二部分，Docker进阶

### Docker Compose && Docker Swarm
- Compose用于部署多个镜像来组成一个应用，通过yaml文件定义需要部署的镜像、网络和卷
- Swarm包含两方面，Docker安全集群和微服务应用编排引擎
- Swarm集群包含多个节点，每个节点需要安装Docker，并且能够互相通信
- Swarm管理节点内置HA支持，使用Raft算法在管理节点中选出领袖节点
- docker service create创建服务，可以指定镜像/数量/端口等参数；docker service scale扩缩容
- docker service update可以更换新的镜像来实现滚动升级

### Docker网络
- Docker网络架构由3部分组成：容器网络模型CNM、Libnetwork(CNM具体实现)、驱动实现特定网络拓扑来拓展模型能力
- 覆盖网络可以允许容器之间方便地直接通信

### Docker Stack
- Stack基于Compose和Swarm，提供了大规模部署和生产环境需要的功能
- Stack yaml文件一般包含以下定义：version代表Compose文件格式版本号；services定义组成应用的服务；networks定义网络；secrets定义密钥
- Swarm支持的参数，例如服务数量和升级参数，都可以在Stack文件内定义
- 所有配置的变更都应该通过Stack文件进行声明，并通过docker stack deploy部署。不要直接用命令行修改，这样再次部署时会回滚到Stack文件的配置

### Docker安全
- 安全机制一般会通过分层来提供，Linux提供了命名空间/控制组/系统权限/强制访问限制/安全计算等安全技术，Docker提供了Swarm模式/安全扫描/内容信任/密钥管理等技术
- NameSpace能够将系统拆分，使得应用可以运行在多个互相独立的系统上
- ControlGroup允许用户设置对每种资源设置限额

