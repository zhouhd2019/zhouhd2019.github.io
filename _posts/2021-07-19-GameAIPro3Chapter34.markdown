---
layout:     post
title:      "1000 NPCs at 60 FPS"
subtitle:   "GameAIPro3 Chapter34"
date:       2021-07-19  10:21:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

Project Highrise是一个模拟经营摩天大楼的游戏，本章讲述了AI方面的性能优化，包括决策和执行两部分

## 决策
- AI主要用来模拟上千个NPC的日常行为
- 决策部分有两种实现，先尝试了命题规划，后面改成了更简单的基于时间的日常行程脚本
- 命题规划由一些规则组成，每个规则有条件、行动和结果
```
Rule:   Preconditions:  at-home & is-hungry
        Action:         go-to-restaurant
        Postconditions: ~at-home & at-restaurant
```
- 命题规划的规则比较简单，因此可以将条件转化为位标记，这样检查是否满足条件就可以通过位运算(与)完成
- 实际上这个游戏需要的是更加规则化的NPC，因此可以使用更简单的日常脚本来实现
```
name "schedule-office.7"
blocks [
    { from 8 to 20 tasks [ go-work-at-workstation ] }
    { from 20 to 8 tasks [ go-stay-offsite ] }
]
oneshots [
    { at 8 prob 1 tasks [ go-get-coffee ] }
    { at 12 prob 1 tasks [ go-get-lunch ] }
    { at 15 prob 0.5 tasks [ go-get-coffee ] }
    { at 17.5 prob 0.25 tasks [ go-visit-retail ] }
    { at 20 prob 0.5 tasks [ go-get-dinner ] }
    { at 20 prob 0.25 tasks [ go-get-drink ] }
]
```
- 日常脚本由两部分组成，blocks描述的是简单的长时间活动，oneshots描述的是特定时间可能进行的活动，增加一些随机差异
- 一般的效用AI不太适合这个游戏的情况，NPC数量太多。还有，调试不直观，要让策划去查看内部数值，并理解为什么NPC在这个时候做这个行为。相比之下，上面的日常脚本要简单得多

## 执行
- 当决策部分给出需要进行的活动，执行部分会将该活动分解为多个子活动，放到队列里面逐个进行。遇到不能执行的子活动，就会清空当前队列，执行保底活动(例如NPC抱怨)，再重新进行决策
- 执行部分不会主动对环境变化做出反应，这样性能消耗较大。另外决策速度很快，重新进行决策也不会有很大负担
- 通用的寻路是将地图分为格子，再通过A*等方法进行寻路。但本游戏是摩天大楼模拟经营游戏，可以进行大量简化
- 本游戏是一个2D游戏，对摩天大楼进行了剖面简化，因此每层寻路可以用多条线段来描述，当楼层发生变化时，重新计算线段
- NPC主要在每层大楼进行活动，上下层通过楼梯、电梯和扶梯进行，
- 寻路首先判断起点和终点是不是同一层，不是则进行层间寻路，寻找终点附近的电梯(楼梯和扶梯)。当移动到终点所在层，再进行简单的直线移动
