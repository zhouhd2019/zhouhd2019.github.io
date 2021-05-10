---
layout:     post
title:      "Context Steering: Behavior-Driven Steering at the Macro Scale"
subtitle:   "GameAIPro2 Chapter18"
date:       2021-05-10  20:40:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro2
    - GameProgramming
---

一种简单易懂的Steering Behavior方法，能够很好地解耦各个相关功能，每个功能可以记录自己的多个期待朝向和拒绝朝向，最后根据实际情况综合朝向

## Steering Behavior
- Steering Behavior包含多个子Behavior来决定本次计算的最终朝向
- 传统实现每个子Behavior可以给出一个朝向，结果由各个子Behavior综合而来
- 传统实现不符合部分子Behavior的需求，例如避让Behavior，目的在于禁止角色往某些方向移动，而不是给出一个建议方向
- 传统实现很难避免耦合和重复计算，例如追逐Behavior，除了考虑目标距离，还要先考虑目标之间到底有没有阻挡，或者用离目标的寻路移动路径，这就和避让Behavior、寻路Behavior等等出现耦合或者重复计算
- 最简单的修正方法是给子Behavior不同的权重，但依然很难避免特殊情况，而且权重需要找到合理值
- 还可以给出优先级，例如避让静态障碍的优先级最高，但这样接近静态障碍的时候就只会僵硬地远离障碍，因为每个子Behavior只能有一个建议方向；
- 还有一种方法，就是具体情况具体解决，例如上述追逐Behavior的实现，这样会有耦合，有时没什么大问题，有时会让代码复杂度大大提高

## Context Steering
- Context Map，就是用一个数组来表示一个角色可以移动的多个方向，例如常见的8方向移动，数组长度就是8，每个元素表示该方向移动的距离
- Context Map有两种，一种是danger map，代表应该禁止的方向；另一种是interest map，代表希望前进的方向
- 追赶Behavior需要一个interest map，对附近每个目标计算方向和距离，越远的目标对应的移动向量应该越短。另外注意，每个目标一般不仅仅对应一个数组元素，一般不会刚好朝向一致，每个目标一般会对应两个元素，要做向量分解
- 避让需要一个danger map，和追赶类似，对附近每个障碍计算移动向量，代表某个方向上离障碍的远近
- 怎样综合Context Map的数据，这应该根据具体游戏的需求而定
- 一般可以这么做，综合所有danger map，每个元素取对应位置的最大值，可行的方向就是那些danger map最小值的位置(一般是0值，会有多个位置)，然后看看那些最小位置对应的interest map上的移动向量，取它们的最大值，作为角色最终移动方向
- 取interest map的移动向量时，还可以计算邻近几个方向的平均值
- 对于赛车游戏来说，可以将目标朝向定为赛道的前进方向，每个元素代表当前位置的某个前进方向。其它游戏也可以根据实际情况来调整计算方向实际意义和数量

## 优化
- 保留上一帧计算的context map，和本次计算的进行插值，这样可以获得一个平缓的过渡，减少反复的来回变化
- 减少计算方向的数量，可以减少计算时间和存储消耗
- 由于每个子Behavior的计算都是独立且无状态的，可以使用多线程进行加速
