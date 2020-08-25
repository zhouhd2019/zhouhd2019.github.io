---
layout:     post
title:      "GameAIPro Chapter11"
subtitle:   "Reactivity and Deliberation in Decision-Making Systems"
date:       2020-08-25 10:50:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameProgramming
---

## Reactivity and Deliberation
- Reactivity是指agent对外界刺激做出反应的能力。Deliberation是指agent根据多种因素决定接下来做什么的能力。

## Common Pitfall 1: One Decision Model to Rule Them All
- 对于Reactivity和Deliberation来说，发起新行为的原因是不一样的。前者通常是由于出现优先级更高的事件，打断当前行为；后者往往是因为老的行为结束了，或者来到了新的环境
- 两者采取的行为也不一样，Reactivity需要采取迅速的反应，而Deliberation会做出长期的规划
- 如果只考虑Reactivity，agent的动画可能会发生抖动和频繁切换

## Common Pitfall 2: one Conceptual Model to Rule Them All
- Reactivity一般和危险感知相关，而Deliberation主要关注程序上的认知
- 决策系统应该加入Action Selector中间层，让它来判断当前action是reaction还是deliberate action，再调用相应模块
