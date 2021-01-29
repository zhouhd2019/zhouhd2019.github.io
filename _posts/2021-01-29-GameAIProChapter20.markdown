---
layout:     post
title:      "Precomputed Pathfinding for Large and Detailed Worlds on MMO Servers"
subtitle:   "GameAIPro Chapter20"
date:       2021-01-29  20:02:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameProgramming
---

## 网格生成
- 寻路网格的自动生成采用Recast来实现
- 主要的修改是在生成过程增加了“滑坡”网格，还有很多小修改：可以用长方体指定一片区域可以或者禁止生成网格；门，用来分割网格和给网格打标记；网格种子点，只有可以和种子点相连的区域才能生成网格
- 玩家可以从陡坡滑下来，并且可以将怪物从坡上撞下去，因此加了滑坡区域功能。这样可以用寻路算法方便计算滑坡的路径和终点，很容易就能保证终点在可行寻路区域

## 连通表格概述
- 寻路网格生成时，Recast会将世界整齐分为多个正方体，每个正方体包含部分网格，定义为tile
- 一个寻路网格多边形被称为一个node，同一个tile内可连通的多个node被称为component
- 每个tile都有两个查找表。一个描述了这个tile内每个node和以这个tile为中心的3X3范围(9个tile)内的每个node的连通性,称之为node连通表。另一个描述以3X3为中心，再往上下左右扩张一次(每个方向3个tile)，新增的12个tile内的每个component和中心tile每个node的连通性，称之为component连通表
- 滑坡网格不需要再连通表里面出现，因为它只能作为单向中间路径，只能从上面滑到下面，中间也不能停，因此在计算连通性时直接处理
- 每次寻路搜索，先看一下在不在node连通表，不在的话就通过更高级别寻路层去搜索
- 为了进一步减少空间消耗，使用三个层次来存储寻路数据。第一个层级是node多边形，第二个层级是component同一个tile内连通多边形，第三个层级是2X2个tile内相连component。

## 连通表格生成算法
```c++
// 连通表格生成伪代码
ComputeConnectivity();
FallingMeshSetup();
BuildMeshTable();
for (int i = 0; i < max_level; ++i) {
    ComponentComputation(i);
    SplitProblematicComponents(i);
    ComputeComponentCenter(i);
    if (i + 1 != max_level) {
        AddHierarchy(i);
        BuildHierarchicalTable(i);
    }
}
for (int i = max_level - 1; i > 0; --i) {
    RemoveInvalidSubLink(i);
}
```
- 计算连通性比较简单，另外还需要输出一些滑坡网格需要的数据
- FallingMeshSetup负责计算滑坡路径，并且设置滑坡输出边界
- BuildMeshTable要计算相邻tile里面的node的连通性，记录经过哪条边到达目标node，这样通过多次查表，就可以知道任意两个node之间的路径，这是初步连通表格
- 如上所述，连通表格应该有两个，数据应该是一部分node2node，一部分node2component，但现在还没有component，所以全部都是node2node。对于每个tile，计算周围20个tile和它的连通数据，邻近8个加上4方向各扩展3个
- 每个tile都有一个初步连通表格，可以多线程处理这一步
- component计算分为两步，第一步是尽可能合并node，第二步是分解有问题的component，简单来说基本是凸边形即可
- component中心的一个可行点是重力点
- RemoveInvalidSubLink用来去除一些过长的路径，它们可能不是最短的路径，去除后，每次多搜索一两步，得到的结果一般更短更好
