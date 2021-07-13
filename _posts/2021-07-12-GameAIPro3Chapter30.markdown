---
layout:     post
title:      "Hierarchical Portfolio Search in Prismata"
subtitle:   "GameAIPro3 Chapter30"
date:       2021-07-12  20:31:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

Hierarchical Portfolio Search(HPS)，用于卡牌策略游戏Prismata。一种让MCTS和Alpha-Beta等算法在比较复杂的游戏中落地的方法，根据游戏实际内容进行划分，再进行决策。例如分成多个阶段，每个阶段给出不同策略，让MCTS等算法来决定使用哪种策略。这样可以简化决策过程，减少AI分支

## 概览
- 现代AI方法需要应用在复杂游戏上，能够应对海量单位和众多决策分支，同时要解决以下问题
- 新玩家教程：策略游戏通常拥有复杂的规则、很多不同单位和不同场景，学习曲线比较陡峭，游戏需要提供不同难度和拥有不同能力的AI
- 单人重玩价值：AI如果每次都用相同的策略，玩家很快就会感到枯燥，要在保证战斗力的同时拥有一定随机性
- 拥抱变化：在线策略游戏经常做出调整，例如新增单位，削弱或者增强单位，AI修改需要比较快速，不应依赖特定数值和单位
- 符合直觉的模块组合设计：AI设计者易于理解，便于修改

## Hierarchical Portfolio Search(HPS)
- 核心思想是减少游戏决策分支，使得能够快速找出足够强的行动。减少分支的方法可以是最简单的手动指定，或者复杂的搜索算法。对于实时策略游戏，通常可以从战术上进行分解，将类似的行动分到同一分支，例如攻击，防守等等
- HPS是一种自底向上的双层搜索系统。底层由一种或多种算法组成，负责对每个游戏部分(阶段或者区域)产生可行建议。顶层是一种高级搜索算法，例如MCTS或者alpha-beta，从可行建议的组合中找出下一步的行动
- HPS不能找到真正全局最优行动，不过一般能符合要求。另外，全局搜索完全不可行，因为可行分支的组合过多
- 对于Prismata，可以将游戏每个回合分为多个阶段，例如防守和进攻，每个阶段有多种不同的行动。可选行动数量要尽量多，这样上层搜索才能找出足够好的行动
- 下面是使用Negamax作为上层搜索算法的HPS的代码示例
```
procedure HPS(State s, Portfolio p)
    return NegaMax(s, p, maxDepth)

procedure GenerateChildren(State s, Portfolio p)
    m[] = empty set
    for all move phases f in s
        m[f] = empty set
        for PartialPlayers pp in p[f]
            m[f].add(pp(s))
    moves[] = crossProduct(m[f]: move phase f)
    return ApplyMovesToState(moves, s)

procedure NegaMax(State s, Portfolio p, Depth d)
    if (d == 0) or s.isTerminal()
        Player e = playout player for evaluation
        return Game(s, e, e).eval()
    children[] = GenerateChildren(s, p)
    bestVal = -infty
    for all c in children
        val = -NegaMax(c, p, d-1)
        bestVal = max(bestVal, val)
    return bestVal
```

## 其它
- HPS的难度控制比较简单，例如改变底层行动的数量，使用不同顶层算法等等
