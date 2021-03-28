---
layout:     post
title:      "Faster Dijkstra Search on Uniform Cost Grids"
subtitle:   "GameAIPro3 Chapter23"
date:       2021-03-28  15:50:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

Dijkstra搜索常用于单源最短路径计算，融合JPS方法的思想，可以加快Dijkstra搜索，本文将这种方法命名为Canonical Dijkstra

## JPS
- A*不会重复搜索已经搜索过的格子，但如果可以直接排除对称的格子，性能会更好
- Canonical Ordering：如果当前格子是对角线格子移动过来的，那么下一步只能朝相同方向的三个格子移动，相同对角线方向和对角线分解的两个正向(N/S/E/W)；如果是正向移动过来的，则下一步只能朝相同的正向；优先对角线方向，没有了再朝正向移动
- 上述规则不能遍历整个地图，因为正向只能接着正向，所以Canonical Order还有一个规则，如果正向经过阻碍，则下一个点就是跳点，可以使用对角线方向

## Canonical Dijkstra
- 使用Canonical Ordering来产生新的搜索点，从起点开始遍历，每个方向到达跳点就停止下来，直到只剩下跳点没有遍历，再从跳点开始。如果某个点之前不是跳点，后面某条路径经过发现是跳点，则需要把它作为跳点，放入open list等待后续遍历
```
CanonicalDijkstra(start)
{
    initialize all states to be in closed with cost inf
    place start in open with cost 0
    while (open not empty)
        remove best from open
        CanonicalOrdering(best, parent of best, 0)
}
CanonicalOrdering(child, parent, cost)
{
    if (child in closed)
        if (cost of child in closed > cost)
            update cost of child in closed
        else
            return
    if (child is jump point)
        if on closed, remove from closed
        update parnet  // for canonical ordering
        add child to open with cost
    else if (action from parent to child is diagonal d)
        next = Apply(d, child)
        CanonicalOrdering(next, child, cost + diagonal)
        next = Apply(first cardnal in d, child)
        CannonicalOrdering(next, child, cost + cardinal)
        next = Apply(second cardinal in d, child)
        CanonicalOrdering(next, child, cost + cardinal)
    else if (action from parent to child is cardinal c)
        next = Apply(c, child)
        CanonicalOrdering(next, child, cost + cardinal)
}
```
