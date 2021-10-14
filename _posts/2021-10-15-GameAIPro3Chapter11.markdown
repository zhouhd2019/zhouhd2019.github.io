---
layout:     post
title:      "A Character Decision-Making System for FINALFANTASY XV by Combining Behavior Trees and State Machines"
subtitle:   "GameAIPro3 Chapter11"
date:       2021-10-15  00:00:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

本篇经验来自FF15，结合行为树和状态机来实现决策系统。

### 概述
- 在逻辑复杂的游戏里，行为树可以嵌套子行为树，或者状态机嵌套子状态机
- FF15实现了一套行为树和状态机互相嵌套的AI决策系统，每个节点可以是子行为树或者子状态机
- 为了让每个节点可以放在行为树或者状态机，每个节点需要实现4个接口
    1. Start process(when a node is called)
    2. Update process(when a node is executed)
    3. Finalizing process(when a node is terminated)
    4. A conditionto signal termination
- 状态机的状态转移一般由外部驱动，而行为树的由节点自行驱动，因此需要将终止条件抽出来单独作为一个需要实现的接口
- Blackboard分为两种，局部的只有角色本身使用，全局则所有角色共享
- 该系统支持并行和打断逻辑。并行节点只能有一个，一般用于简单持续行为。打断行为可以有多个，符合条件则触发
- Prefab功能使得每个角色可以引用某个AI文件，然后自己做出一些小修改
- 每个AI角色的目标由角色自身的探测系统来负责更新，这样不需要每个AI节点重复搜索
