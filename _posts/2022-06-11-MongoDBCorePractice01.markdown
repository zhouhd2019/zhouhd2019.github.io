---
layout:     post
title:      "MongoDB基础知识"
subtitle:   "MongoDB核心原理与实践1"
date:       2022-06-11  16:22:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - MongoDB
    - DataBase
---

MongoDB核心原理与实践摘录，第1篇MongoDB基础知识

### 初识MongoDB
- mongod，数据库实例守护进程，运行在服务端
- mongo，与mongod交互的javascript shell进程
- mongos，用于客户端操作分片路由

### CRUD
- sort/skip/limit一起使用时，无论语句顺序是什么，执行顺序都是sort/skip/limit
- 插入操作对于单条文档记录来说是原子性的，不会有插入一半的问题
- insertMany/insert插入多条记录时，如果有一条记录发生错误，则不会有任何记录被插入，插入过程是一个整体
- bulkWrite用于批量写操作，它的ordered参数默认为true，表示操作按顺序执行，出错时停止执行；如果设置为false，则每条单独执行，互不影响。注意同一批写操作中不能对同一个文档执行多次操作

### 索引
- MongoDB支持数组索引，默认对数组每个元素创建索引条目，通过索引条目可以找到包含这个元素的数组的文档
- explain可以不传参数，如果传executionStats或allPlansExecution可以获得详细计划信息，例如db.colName.explain("executionStats").find(...)
- PROJECTION_COVERED表示本次是索引覆盖查询，需要的字段包含在索引中
