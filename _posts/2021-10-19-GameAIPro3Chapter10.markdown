---
layout:     post
title:      "From Behavior to Animation: A Reactive AI Architecture for Networked First-Person Shooter Games"
subtitle:   "GameAIPro3 Chapter10"
date:       2021-10-19  20:52:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

行为树的一个简单实现，另外描述了如何做一个简单的动画状态机，没有什么特别的地方

### 概述
- 一般用Blackboard存放数据，行为树本身只包括逻辑
- Blackboard除了一般的值类型，还有一类函数类型，实际值是一个update_function，调用这个函数获得最新的数值
- 并行节点不要处理过于复杂的逻辑，一般是一个行为节点加上一个不断运行的条件节点，条件失效时停止运行
- 要支持并行节点，行为树需要记录当前正在运行的所有节点，而不是一个节点
