---
layout:     post
title:      "Being Where It Counts: Telling Paragon Bots Where to Go"
subtitle:   "GameAIPro3 Chapter24"
date:       2021-06-04  10:49:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

虚幻争霸用到的一种方法，将地图简化为一个图，每个节点代表地图上有意义的位置，每条连接代表节点之间可行路径，用这个图来计算AI角色应该到哪个位置。不过，现在MOBA已经以人工神经网络为主了

## 战略图
- MOBA游戏一般存在两个决策层面：战术层面，负责具体某个机器人的行为；战略层面，基于战局情况对整个队伍作出指示。本文主要关注战略层面的AI设计
- 战略层面AI需要解决机器人去哪个地方的问题，到了所需位置，剩下的就是战术层面的问题
- 首先需要定义所有可以去的地方，MOBA游戏里主要包括基地、守护塔、野怪点、有利地形位置、埋伏点等等，它们都是战略图里面的节点，一部分是地图本身元素，一部分是所谓的战略地点
- 节点需要记录位置、地图相关元素的索引和在哪条路上等等属性
- 对于节点连接，虚幻争霸采用了暴力遍历方法，让每个节点连接其它所有节点。接着将多余的连接标记出来，如果一条连接可以被其它多条连接代替，即移动消耗更多，那这条连接就是多余的。注意不是删除，后续可能需要
```
Graph::PruneEdges(InNodes, InOutEdges)
{
    SortEdgesByCostDescending(InOutEdges)
    for Edge in InOutEdges:
        for Node in (InNodes - {Edge.Start, Edge.End}):
            if Edge.Cost >= (InOutEdges[Edge.StartNode][Node].Cost + InOutEdges[Node][Edge.EndNode].Cost + EdgeCostOffset):
                    Edge.IsPruned = true
                    break
}
```
- 最近节点查找，可以通过预先计算查找表来快速获得结果

## 战略图：非AI应用
- 战略图部分大部分数据通过离线计算获得，一些数据例如危险度则需要游戏内实时更新
- 危险度一般需要计算所有英雄的位置和所有小兵的位置
```
Graph::UpdateInfluence(InAllHeroes, InAllMinionWaves)
{
    ResetInfluenceInformation()
    for Hero in InAllHeroes:
        Node = LookupClosestGraphNode(Hero.Location)
        Influence = CalculateHeroInfluence(Hero, 0)
        Node.ApplyInfluence(Influence)
        for Edge in Node.Edges:
            Influence = CalculateHeroInfluence(Hero, Edge.Cost)
            Edge.End.ApplyInfluence(Hero.Team, Influence)

    for Wave in InMinionWaves:
        Node = LookupClosestGraphNode(Wave.CenterLocation)
        Influence = CalculateWaveInfluence(Wave, 0)
        Node.ApplyInfluence(Influence)
        for Edge in Node.Edges:
            if Edge.EndNode.LaneID != Wave.LaneID:
                continue
            Influence = CalculateWaveInfluence(Wave, Edge.Cost)
            Edge.End.ApplyInfluence(Wave.Team, Influence)
}
```
- 注意上述危险度计算，不会排除已经被标记为多余的边，应用危险度时会考虑到距离，对于很远的点，影响很小
- 寻路移动时，应该充分考虑战略图的数据，例如要躲避敌人，就不应该走那些危险度高的路径
- 节点间移动可以先搜索出中间途径的节点，逐个移动，这样可以大大减少路径搜索时间

## 战略图：AI应用
- 战略图里面每个节点都有一定的战略价值，战略目标主要包含一个战略图的节点和其它所需数据
- 战略目标应该知道：如何处理该位置(防守、进攻等等)，需要多少个英雄，每个英雄执行该战略目标的适合度
- 不同的战略目标有不同的优先级，并且这个优先级会随着游戏实时变化
- AI管理器负责管理战略目标，例如更新信息和分配目标给机器人

## 战略目标分配
- 第一步，每个战略目标更新自身信息，计算优先级，状态变为休眠或者有效，休眠表示目标现在不能执行。对战略目标根据优先级进行排序
- 第二步，每个有效战略目标针对每个英雄计算适合度
- 第三步，分配目标：先找到优先级最高的目标，然后让这个目标选择基本满足条件的最少资源(主要是英雄)，根据选择的资源来减少这个目标的优先级。循环上述步骤，直到每个英雄都有战略目标可以执行
- 上述第三步每次使用最少资源，这可以根据游戏实际需求更改
- 战略目标优先级和适合度这两种数值的计算方法，需要好好调整，影响到实际的全局表现
- 对于每个战略目标，它评价英雄的适合度方法有两种，一种是完全自定义，另一种是指定通用模板，使用几个重要参数根据某个公式给出结果
- 英雄职能和离战略目标的距离是计算适合度的常用参数。距离有时不准确，到达目标消耗时间可能更好
- 对于玩家，AI管理器同样进行计算，并向玩家给出一定提示。计算时也可以考虑玩家给出的提示。另外如果玩家没有按照预期行动，也没有关系，下次计算依然正常执行
- 当机器人前往执行某个战略目标，寻路算法可以考虑给出一条可以顺路执行其它简单任务的路径，例如顺路打野

## 未来工作
- 位置概率图：如果敌人不可见，基于战略图，参考它上次出现的地方、它可能去往的战略目标等等信息，对敌人位置进行猜测
- 人工神经网络，将已有信息作为输入，获得当前战略图一些信息的估计值
