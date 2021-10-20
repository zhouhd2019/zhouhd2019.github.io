---
layout:     post
title:      "Overcoming Pitfalls in Behavior Tree Design"
subtitle:   "GameAIPro3 Chapter09"
date:       2021-10-20  10:52:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

行为树实现和应用的经验之谈

### 常见问题
- 增加新的基础节点类别。行为树基础节点包括行为、装饰和复合节点。如果要增加新的基础节点类别，会影响整个系统的实现。基础节点越少，系统实现和理解成本越低
- 过多的逻辑相关条件节点。如果逻辑比较复杂，条件节点可能的确需要很多种。但应该尽量复用已有的节点，给已有节点增加接口，新节点改写接口
- 所有东西通过黑板来交流。这样会使得逻辑过于依赖黑板，黑板本身应该只有通用数据
