---
layout:     post
title:      "Using Your Combat AI Accuracy to Balance Difficulty"
subtitle:   "GameAIPro3 Chapter33"
date:       2021-07-16  20:48:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

非常简单的一篇文章，射击游戏如何通过调整AI的准确度来调整难度，主要是一些细节

## 令牌算法
- 调整AI难度的方法有很多，属性、反应速度等等都可以，本章主要讲的是射击游戏里面调整AI的准确度
- 令牌算法，一种常见的用于服务器限流的算法，也可以用于AI，只有获得令牌的AI可以击中玩家，其它情况下AI可以射击，但不会准确瞄准玩家，很难击中
- 令牌发放的时间间隔可以由多个因素决定，这里用的乘法公式，基础的间隔乘以其它各个因素的因子，得到实际间隔
- 常见令牌因素有距离、玩家朝向、玩家姿势等等
- 如果面对多个AI，可以独立计算令牌间隔，并根据数量适当调大间隔。也可以使用一个定时器定期更新，自行决定接下来令牌给哪个AI
- 当AI不能准确瞄准玩家时，可以选择玩家附近的位置进行射击，如果附近有可破坏物，射击它们可以大大提升游戏真实感
