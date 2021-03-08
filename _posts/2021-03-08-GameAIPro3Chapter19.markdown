---
layout:     post
title:      "RVO and ORCA: How They Really Work"
subtitle:   "GameAIPro3 Chapter19"
date:       2021-03-08  11:51:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

## 概述
- 最早的VO算法很容易出现震荡，角色反复修改自己的朝向
- RVO让两个角色各负责一半的避让，找出最佳不碰撞轨迹。但多角色下可能会有问题。一般会实现为持续的避让，即这一帧各自避让，下一帧发现以后依然可能碰撞的话，就进一步避让

## ORCA(RVO2)
- ORCA一般要求角色保持方向性，即一直保持朝左或者朝右避让。这样在转弯的时候可能有问题，需要另外处理
- 可以将VO理解为各自负责权重为1的避让，RVO就是各自0.5。权重越大，避让越明显越快速，但可能出现震荡。权重小则避让慢，可能出现碰撞
- 上述可以知道RVO系列算法可能需要多帧才能避让成功，实际上也可以一帧内多次迭代
- 转角问题可以简单地处理，只考虑转弯前的避让，转弯后再去计算
