---
layout:     post
title:      "Phenomenal AI Level-of-Detail Control with the LOD Trader"
subtitle:   "GameAIPro Chapter14"
date:       2020-09-12 23:38:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameProgramming
---

## AI LOD
- 不仅仅决定AI决策频率，还可以包括寻路准确度和IK等等

## 失真
- 失真主要有三种，非真实状态/基础中断/失真长期行为，不同的失真有不同的关键因素
- 对于非真实状态，即玩家观察到NPC状态不正常，例如朝着墙跑。关键因素是可观察性（画面远近）和注意力（玩家注意到NPC的程度）
- 基础中断指一些细节，NPC当前状态和玩家之前所知状态不匹配，例如NPC断腿了还在奔跑。关键因素是记忆力（玩家能记住NPC当前多少状态）和返回时间（玩家再次见到NPC时记忆是否清晰）
- 失真长期行为是指角色进行了很长一段时间的有问题行为，例如NPC无意义闲逛。关键因素是注意度、记忆和持续时间，其中持续时间指玩家在NPC身上所花的时间

## 关键性模型
- 可观察性，角色占屏幕空间越多像素越多，就越高。用p表示占屏比例，O=min(p/pmax, 1)
- 注意力，最难建模的参数，和游戏类型高度相关，可以参考角色重要程度/速度等等。玩家注意力是有限的，可以把所有角色的注意力参数加起来做一个归一化
- 记忆力，和注意力有关，随着时间逐渐减少
- 返回时间，输出结果是当玩家再次看到某个角色时的记忆衰减参数
- 持续时间，和可观察性和注意力相关

## LOD Trader算法
- LOD Trader算法记录所有角色当前的LOD，每次计算在有限计算资源下，找出失真较小的LOD方案
- 每次计算，LOD Trader评估当前方案，在多个候选方案中作出决策。部分角色的相对关键性评分变化了，从而它的LOD需要调整
- 不考虑消耗的话，决定某个角色的LOD方法是，将角色关键性评分向量（上述模型是三维）和LOD的评分向量相乘，选择点积最小的那个LOD
- 寻找可用方案的初始算法实现如下
```py
def runLODTrader(characters, lodLevels, availableResource):
    acceptedTrades = []
    while True:
        # returns p-queues, sorted by value
        upgrades, downgrades = calcAvailableTrades(characters, lodLevels)
        hypTrades = []
        charactersWithTrades = []
        hypBenefit = 0
        hypAvailableResource = availableResource
        while not upgrades.empty() and availableResource > 0:
            upgrade = upgrades.pop()
            hypTrades.append(upgrade)
            charactersWithTrades.append(upgrade.character)
            hypAvailableResource -= upgrade.costIncrease
            hypBenefit += upgrade.probDecrease
        while not downgrades.empty() and availableResource < 0:
            downgrade = downgrades.pop()
            if downgrade.character in characterWithTrades: continue
            hypTrades.append(downgrade)
            characterWihtTrades.append(downgrade.character)
            hypAvailableResource += downgrade.costDecrease
            hypBenefit -= downgrade.porbIncrease
        if hypAvailableResource >= 0 and hypBenefit > 0:
            acceptedTrades += hypothesizedTrades
            availableResource = hypAvailableResource
        else
            return acceptedTrades
```
- 一般会有多个AI相关系统需要LOD，例如寻路和IK。如果每个系统单独跑一个LOD Trader，起码会有两个问题。资源限制是全局的，计算时需要考虑资源给各个系统的分配，单独跑很难做到这点。另外，各个系统间可能会有依赖
- 应该使用一个LOD Trader来同时计算和平衡多个系统之间的LOD，每次计算需要找出多个系统的LOD，这可以使用向量表示
```py
def expandCharacter(char, transType):
    bestRatio = None; bestTrans = None
    for trans in char.featureSolution.availableTransitions[transType]:
        ratio = dotProduct(char.C, trans.A)/trans.cost
        if isBetterRatio(ratio, bestRatio):
            bestRatio = ratio; bestTrans = trans
    return bestRatio, bestTrans
```
- 多个系统LOD的同时计算消耗太大，可以加入一个角色优先队列，根据角色可能达到的最佳值相关的数值来排序（可以分开上升和下降LOD两个队列）。此时可以采用不用策略来决定怎么处理，例如每个角色只处理一次，队列为空时重新填充
- 还可以通过剪枝来减少计算量，例如某个系统用最低LOD，其它系统用最高LOD，这样的方案不太可能出现。另外在上升队列，可以提前排除那些所有系统LOD都不大于当前最佳方案的方案，下降队列同理
- 除了CPU，内存占用量也是需要考虑的资源，因此算法应该支持多种资源