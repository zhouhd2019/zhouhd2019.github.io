---
layout:     post
title:      "Subgoal Graphs for Fast Optimal Pathfinding"
subtitle:   "GameAIPro2 Chapter15"
date:       2021-04-21  20:40:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro2
    - GameProgramming
---

简单来说，这个方法有点类似JPS，但相对简单一些，性能没那么好

## 概览
- Subgoal Graph通过预处理格子地图，找出关键点，搜索时主要关注关键点，从而加快搜索速度
- octile距离是两个格子间最短路径的距离，也就是实际移动的距离，适合作为启发因子用于A*等搜索方法
- octile距离计算时不考虑阻碍，如果x大于y，则结果为y + (x - y) * sqrt(2)

## Simple Subgoal Graph
- SSG由可见性图改编来用于格子地图，每个节点都在障碍的凸多边形顶点上，节点之间按照一定规则连接得到边，边的长度是节点间的octile距离
- 如果一个格子s本身可以通行，有一个对角障碍邻居格子t，而且s和t之间的两个邻居格子可以通行，那么s就是一个Subgoal
- 如果两个格子之间的最短路径长度等于启发因子，那么这两个格子符合h-reachability
- 如果两个格子符合h-reachability，但其中一种最短路径经过了另一个Subgoal，这个Subgoal和它们符合h-reachability，那么原来的两个格子之间不需要连接，这条连接是多余的，应该只保留direct-h-reachable的格子连接
- 简单来说，SSG的顶点是障碍拐角(凸多边形顶点)，边是direct-h-reachable的Subgoal连线
- Clearance(s, c)的结果是格子s朝c方向检测，到下个障碍或者Subgoal的距离
- 找出一片区域的所有direct-h-reachable Subgoal的伪代码，注意在这之前就知道哪些格子是Subgoal
```
GetDirectHReachable(cell s, cardinal dir.c, diagonal dir.d)
    SubgoalVector list = {};
    int maxLineLength = Clearance(s, c);
    int nDiaMoves = Clearance(s, d);
    for int i = 1 ... nDiagMoves
        s = neighbor of s toward d;
        l = Clearance(s, c);
        if (l < maxLineLength)
            maxLineLength = l;
            _s = the cell l+1 moves away from s toward c;
            if (_s is a subgoal)
                list.add(_s);
    return list;
```

## Two-Level Subgoal Graphs
- SSG的基础上，将Subgoal分为局部Subgoal和全局Subgoal，只有全局Subgoal的图和所有Subgoal的图都是TSG的一层
- 从SSG构建TSG的伪代码
```
ConstructTSG(SSG S)
    SubgoalList G = subgoals of S;  // Global subgoals
    SubgoalList L = {}; // Local subgoals
    EdgeList E = edges of S;
    for all s in G
        EdgeList E+ = {}  // Extra edges
        bool local = true; // Assume s can be a local subgoal
        for all p, q in Neighbors(s)  // Neighbors wrt E
            d = length of a shortest path between p and q (wrt E) that does 
                not pass through s or any subgoal in L;
            if (d > c(p, s) + c(s, q))
                if (p and q are h-reachable)
                    E+.add((p, q));
                else // s is necessary to connect p and q
                    local = false; // Can't make s local
                    break;
        if (local)
            G.remove(s);  // Classify s as a local subgoal
            L.add(s);
            E.append(E+); // Add the extra edges to the TSG
    return (G, L, E)
```
- 搜索就是简单的先搜索Global Subgraph，找到路径后在SSG继续搜索细化路径
- 参考Two Level，还可以继续进行迭代，得到N-Level Graphs
