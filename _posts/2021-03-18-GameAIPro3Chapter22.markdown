---
layout:     post
title:      "Fater A* with Goal Bounding"
subtitle:   "GameAIPro3 Chapter22"
date:       2021-03-18  20:16:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

## 概述
- Goal Bounding是一种可以大大优化A*等算法性能的技术，而且能够应用在路点、网格和格子等等地方
- 通过离线预计算，Goal Bounding排除了部分不可能到达终点的节点，大大减少了需要搜索的节点
- Goal Bounding有三个限制：地图必须静态不能改变，内存需求较高，需要离线预计算
- Goal Bounding有三个好处：可以应用在任何图搜索空间，包括四叉树、八叉树等等；可以应用在任意图搜索算法，例如Dijkstra和JPS+(优化表现优异)；节点能够有不同的消耗

## 实现
- 预计算简单来说，就是对每个节点(例如导航网格上的每个多边形、路点的每个点)的每个可行方向进行宽度优先搜索，类似洪泛算法，这样每个方向都有一堆可达节点，而且这些节点不重复，是最优可达。对这些节点求包围盒，这样对于每个节点，它的每个方向上有两个坐标来表示包围盒
- 搜索流程大致如下，关键在于WithinBoundingBox，如果这个方向不能到达终点，可以直接排除
```
procedure AStarSearch(start, goal) {
    Push (start, openlist)
    while (openlist is not empty) {
        n = PopLowestCost(openlist)
        if (n is goal)
            return success
        foreach (neighbor d in n) {
            if (WithinBoundingBox(n, d, goal)) {
                StandardAStarOp(...)
            }
        }
        Push(n, closedlist)
    }
    return failure
}
```
- Goal Bounding实际上和Floyd-Warshall任意两点最短路径有点相似，后者需要大量内存空间，1000X1000个节点大概需要4TB空间，而Goal Bounding需要60MB
