---
layout:     post
title:      "Creating High-Order Navigation Meshes through Iterative Wavefront Edge Expansions"
subtitle:   "GameAIPro Chapter19"
date:       2021-01-27  16:37:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameProgramming
---

## Wavefront Spatial Decomposition
##### 初始播种
- 传统上，基于生长的算法使用基于格子的模式，在世界摆放单位大小的初始区域。接着遍历每个区域进行生长和扩张操作
- Wavefront算法首先生成潜在种子点列表，将种子放在开放的障碍边。这样可以更好地覆盖整个环境，而且比传统方法需要更少的初始区域

##### 边界分类
- 每次扩展一个区域，并且不再处理那些已经处理过的种子
- 扩展时，遍历世界内障碍和区域的每条边。然后去掉那些法线背向当前区域种子点的边，因为它们不能和当前区域相连
- 接下来根据边和种子点的相对位置进行分类，三维空间会分成8个象限（二维4个）
- 分类完成后，找出所有潜在事件点，即障碍边界上那些可能和扩张后的区域相交的点（可能是端点也可能是边上的点）

##### 边界扩张
- 根据上一步得到的事件点，从近到远尝试扩张，区域在这个过程中需要增加新的边，保持区域是凸边形
- 扩张可能和其它区域发生碰撞，此时应该停止这个方向的扩张

##### 重新播种
- 如果初始种子点都遍历过了，接下来需要对当前已有区域的未处理邻接区域进行尝试处理
- 如果所有区域都处理过了，结束算法

## 对比
- 传统基于扩张的空间分解算法需要进行合并区域等后处理，Wavefront算法每次只扩张一个区域，一般不需要合并
- Wavefront算法能产生数量更少的区域，并且更少接近退化的区域
