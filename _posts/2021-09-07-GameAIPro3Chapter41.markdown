---
layout:     post
title:      "Leveraging Plausibility Orderings to Achieve Extremely Efficient Data Compression"
subtitle:   "GameAIPro3 Chapter41"
date:       2021-09-07  16:46:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

充分考虑数据的背景和特点，来获得足够的压缩率

- 空当接龙存在部分牌局没有解，因此本游戏提前生成了大量可行牌局，并根据难度等条件进行分级
- 牌局和解法如果直接保存，需要的空间比较多
- 首先可以使用Bit Packing来压缩。例如一步操作只有10种，可以用4个位来表示，还有52张卡牌只需要6位
- 对于大量牌局来说，操作可以通过查表来压缩
- 摩斯/霍夫曼编码可以进一步压缩操作码，对于常见的操作，使用更少的位来表示
- 行程编码可以有效压缩连续相同操作
