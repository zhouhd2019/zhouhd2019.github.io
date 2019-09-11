---
layout:     post
title:      "GameAIPro Chapter07"
subtitle:   "Real-World Behavior Trees in Script"
date:       2019-09-11 16:57:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameProgramming
---

   - 行为树执行伪代码，从根节点开始，递归决定要执行的正确分支行为

```
process_behavior_node(node)
    if (node.precondition returns true) {
        node.action()
        if (node.child exists)
            process_behavior_node(node.child)
    } else {
        if (node.sibling exists)
            process_behavior_node(node.sibling)
    }
```

   - 行为树用脚本实现，需要注意垃圾回收，最好能够在确定时刻预先进行垃圾回收，减少不确定导致的顿卡。对于消耗大的运算，例如三角函数和几何运算，可以改用C++来实现
   - AI系统可以加入LOD，玩家看不到的可以少执行更新
   - 如果能够知道某个时刻在运行的节点，会使得调试更加方便和准确。实现这个功能最简单的方法是使用位图，当然这要求行为树在运行时是不变的，将位图每个位和行为树的节点一一对应起来，这样就可以记录任意时刻正在运行的节点
