---
layout:     post
title:      "Efficient Crowd Simulation for Mobile Games"
subtitle:   "GameAIPro Chapter24"
date:       2021-02-24  09:34:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameProgramming
---

## 手机游戏的群体仿真
- Fieldrunners 2是一个塔防游戏，利用了流场来优化大量单位的移动
- 场景被离散化为格子，格子比所有单位都要大，因此每个单位都可以直接在格子上移动，不需要考虑半径
- 给定多个格子和多个终点，得到的流场最重要的信息是每个格子的最佳方向，就是这个格子上的单位该朝哪个方向移动来到达最佳的终点
- 流场的优点在于不需要每个单位自己计算，可以在任何位置任何时刻知道应该朝哪个方向移动
- Fieldrunners 2计算流场的算法基于Dijkstra's算法，简单来说是将终点格子放入队列，每次取一个格子，设置或者更新它周围格子的朝向，然后将那些有新数值的格子放入队列，一直这么计算直到队列为空
- 流场的朝向并不是单位最终朝向，计算时根据周围的格子来进行插值，另外需要综合避让等功能来修改速度
- 流场不直接存放朝向，而是存放y轴偏移角度，可以减少内存消耗
