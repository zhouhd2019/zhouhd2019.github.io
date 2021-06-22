---
layout:     post
title:      "The Role of Time in Spatio-Temporal Reasoning"
subtitle:   "GameAIPro3 Chapter27"
date:       2021-06-22  16:39:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

时空推理经常用来解决放置物件的问题，例如战争中在哪里设置火力网，城市在哪里铺设高速路等。分析空间因素的著作很多，本文关注相关资料比较少的时间因素。以塔防游戏为例，分析了在时空推理中的时间因素，同时简单地分析如何写塔防游戏的AI，选择摆放箭塔的最佳位置

## 最大化有效范围策略
- 塔防游戏里面，将塔放置在U型拐弯可以让箭塔攻击更多的范围，这就是最大有效范围策略的一个应用
- 在攻击速度一定的情况下，箭塔攻击覆盖范围越广，能够攻击的次数越多，攻击总时间越长，能够击杀越多怪物，游戏得分越高。这个看起来是个空间问题，实际上也是一个时间问题
- 可用性(Affordance)是指一个事物让人明显知道如何使用的程度，即直观功能，例如门上的把柄暗示门可以拉开
- 最大化有效范围示例代码如下
```c++
Solution AI::getSolution(MapModel map) {
    SolutionRequest request = new SolutionRequest(map);
    GroupToPlace group = new GroupToPlace();
    request.groups.Add(group);
    group.towers.Add(new AttackTower());
    Strategy strategy = new PlaceTowerOnRelativePropertyStrategy(
        MapPropertyOrdinal.PhysicalPathCellInRange, 
        StrategyOperator.Maximum
    );
    group.placementStrategies.Add(strategy);
    return Solver.getSolution(request);
}

static Solution Solver::getSolution(SolutionRequest request) {
    Solution solution = new Solution();
    solution.map = MapAnalyzer.getMapAnalysis(request.map);
    foreach(GroupToPlace group in request.groups) {
        foreach(Tower t in group.towers) {
            foreach(Strategy s in group.strategies) {
                List<GridPoint> candidates = s.getPositions(t, map);
                if(candidates.Count > 0) {
                    GridPoint p = group.tieBreaker.get(candidates);
                    solution.add(t, p);
                    break;
                }
            }
        }
    }
    return solution;
}
```

## 空间对称和差别减速策略
- 上述U型拐角的情况将时空问题转化为了一个空间问题，但面对空间类似的情况，难以进行转化
- 面对两条路径基本一致但长度不同的情况，它们空间类似，但时间因素不一样
- 路径长度差异，这也是一种可用性
- 可以在路径后段放攻击箭塔，不会同时面对过多敌人
- 可以通过减速箭塔来增大差异，使得后续箭塔获得更多攻击时间
- 对每种可用性，AI系统使用一个影响图来记录相关决策数据，对每个格子进行打分，再融合多种可用性评分，得分越高的格子就越适合摆放箭塔

## 量化时空和攻击窗口
- 最大化箭塔攻击时间和时间因素相关，摆放箭塔和空间因素相关，需要一个度量标准来同时衡量时空
- 本章提出AL(agent length)来统一时空度量，例如某个箭塔有6AL激活时间，就是说经过这个箭塔的路径长度为6格，或者说箭塔有6秒用于攻击
- 空间攻击窗口是指箭塔可以攻击的一段连续格子。时间攻击窗口是指箭塔可以攻击的一段时间，总和是箭塔的激活时间
- 前面所说的最大化有效范围，实际上是最大化攻击窗口。如果两队怪物同时经过一个箭塔，攻击窗口也只有一个
- 一列怪物通过一个箭塔，时间攻击窗口大小大致为队伍长度加上可攻击路径长度
- 同一个箭塔连续的两个攻击窗口并不能简单相加，因为怪物队列长度可能横跨两个攻击窗口，AI策略要处理好
