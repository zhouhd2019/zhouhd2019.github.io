---
layout:     post
title:      "Advanced Techniques for Robust, Efficient Crowds"
subtitle:   "GameAIPro2 Chapter17"
date:       2021-05-10  11:50:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro2
    - GameProgramming
---

一般的寻路实现会使用AStar加上局部避让算法。本文考虑到了地图的拥挤程度，在AStar搜索过程中对拥挤的地方给予更高的惩罚/消耗

## 概览
- AStar加上局部避让算法的传统移动实现方案，在角色较多且起点终点相近的情况下，大部分角色都会走同一条路径，忽略了其它可能，挤成一团
- Congestion Maps是一种简单衡量地图拥挤程度的方法，结合Vector Flow Fields，可以很好处理大量角色移动造成的拥挤问题，让它们懂得选择角色少的路径

## 细节
- 首先要计算累计群体密度。记录每个角色的速度和位置，用位置构建Influence Map，得到每个点的角色聚集密度。对速度进行同样处理，计算每个点的平均速度。上述两个图就是所谓的Congestion Map
- 路径搜索时，角色经过某个点，要考虑它的移动方向和Congestion Map的方向，两者点乘再乘以聚集密度值，得到该点的惩罚，伪代码如下：
```c++
float congestionPenalty(Vec2 ideal, Vec2 aggregate, float density)
{
    //Projection of aggregate onto ideal, represented
    //as a scalar multiple of ideal.
    float cost = Vec2.dot(ideal, aggregate);
    cost /= ideal.mag() * ideal.mag();
    //If cost is > 1, the crowd is moving faster along the
    //ideal direction than the agent’s ideal velocity.
    if (cost >= 1) return 0.0f;
    //Cost is transformed to be positive,
    //and scaled by crowd density
    return (1 - cost) * density;
}
```
- 如果有路径平滑，需要注意不要把一些绕路点删掉，它们很可能是用来避让人群的
- 实际移动时还需要局部避让算法
- 计算时需要特别考虑拥挤程度的变化，有些地方可能等到过去的时候就不拥挤了。为了减少来回走动，可能需要延迟或减少计算，还可以等拥挤程度变化较大时才重新计算
- 对于非格子地图，需要自行设计Congestion Maps的粒度和计算方式
