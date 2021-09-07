---
layout:     post
title:      "Vintage Random Number Generators"
subtitle:   "GameAIPro3 Chapter40"
date:       2021-09-07  14:46:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

随机数相关知识

## rand()
- rand()函数的实现基于线性同余随机数生成器(LCG)，快且简单，但产生的随机数不够好
- rand()函数的实现理论基础，决定它如果不好好选择参数，出现周期。如果用来做随机布尔值，还会出现一长串相同值
- 最简单的改进是使用已知能够产生比较好随机数列的参数组合
- 将两个LCG组合成一个，可以产生更好的结果，这样的结果一般来说已经足够好
