---
layout:     post
title:      "Petri Nets and AI Arbitration"
subtitle:   "GameAIPro3 Chapter29"
date:       2021-07-06  10:51:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

非常简单的一篇文章，讲了Petri Net和在游戏中的应用，可以用来控制AI资源分配

## 概览
- Petri Net是一种图，常用于描述分布式系统的信息流，有四种类型组成，place/transition/arc/token
- place通过arc和transition相连，相同类型不能直接相连
- place可以持有token，表示place的条件已经达成
- AI里面一般涉及到资源管理，如果每种AI都自行检查和申请资源，会导致逻辑重复，此时可以增加一个仲裁者来管理资源的申请和分发，仲裁者的具体逻辑可以参考Petri Net来实现
