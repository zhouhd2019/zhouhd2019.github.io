---
layout:     post
title:      "Hierarchical Plan-Space Planning for Multi-unit Combat Maneuvers"
subtitle:   "GameAIPro Chapter13"
date:       2020-09-07 00:24:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameProgramming
---

## Planning for Multiple Units
- 多单位规划需要考虑多单位同时行动，而且可能需要互相协调。常用的规划方法如HTN和GOAT需要对每个单位进行规划，因此不能用于多单位，计算量太大
- 多单位规划一个可行方法是将目标任务分解成（有序或无序）的多个子任务，一直划分直到单位可以直接执行

## Hierarchical Planning in Plan-Space: The Ingredients组成
- Planner Main Loop: 执行(一般是A*)搜索，直至找到可行任务计划
```
loop
    current = get most promising plan from open list
    brek if current.complete? or current.null?
    add current to closed list
    pick t = current.task_to_detail
    for every method m that applies to task t
        alternatives = m.generate(current, t)
        for every a in alternatives
            plan = clone current
            // refine using method m and alternative a
            m.plan(plan, t, a)
            compute plan's cost
            add plan to open list
```
- A Plan of Tasks: 一个计划由多个独立任务组成。一个任务表示由一个或者多个单位执行的活动，也可以划分为多个子任务。例如一个排的行军可以由多个班的行军组成，一个班的行军则由多个单位执行。一般需要定义任务的条件（输入）和结果（输出）
- Planner Methods: 负责细化计划和任务，设法满足任务条件，落实任务结果。具体来说，对于复合任务，规划方法负责将它分解为子任务，找到可行分解方案；对于子任务来说，规划方法负责满足它的条件，以及归纳任务结果。简单来说，这是规划循环的具体逻辑
- Plan-Space: 存放结果，包括最终的和中间结果。简单来说就是规划循环A*搜索时的open list和closed list内容，和最后得到的计划

## Making Planning More Efficient
- 标准HTN没有引入COST，如果加入COST，即使用A*算法，搜索会快一些
- 尝试先分解那些大任务，这样可以更快，因为大任务对于可行性的影响更大，可以快速排除那些不可行分支
- 组合使用前向搜索和后向搜索，尤其是先处理那些满足更多条件的任务，同样可以大量剪枝