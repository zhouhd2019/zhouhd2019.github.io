---
layout:     post
title:      "Theta* for Any-Angle Pathfinding"
subtitle:   "GameAIPro2 Chapter16"
date:       2021-04-25  20:30:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro2
    - GameProgramming
---

简单来说就是修改A*的g(x)计算方法，在计算路径的同时，尽量使得路径平滑。不仅可以用于格子，也可以用于网格

## 概览
- A*逐个格子进行搜索，而且只会用格子中心或者顶点进行搜索，因此容易产生曲折的路径
- 要让路径变得平滑，可以使用一些后处理方法，对路径进行平滑。也可以先获得路径经过的格子，再用拉绳算法求解路径
- 本文方法修改了A*的g(x)计算方法，在计算路径的同时，尽量使得路径平滑，修改灵感来源于可见图
- 下面是修改部分的伪代码，c(x,y)是x到y的直线距离
```
ComputeCost(s, s`)
    if lineofsight(parent(s), s`) then  //如果两点可以直连，就直接计算，不经过中间点
        if g(parent(s)) + c(parent(s), s`) < g(s`) then
            parent(s`) := parent(s)
            g(s`) := g(parent(s)) + c(parent(s), s`)
    else
        if g(s) + c(s, s`) < g(s`) then //常见的A*算法
            parent(s`) := s
            g(s`) := g(s) + c(s, s`)
```
