---
layout:     post
title:      "Combining Scripted Behavior with Game Tree Search for Stronger, More Robust Game AI"
subtitle:   "GameAIPro3 Chapter14"
date:       2021-10-11  20:13:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

本篇主要讲如何将树搜索算法应用到AI决策中

### 概述
- AI脚本可以完成游戏逻辑大部分工作，例如RTS里面造建筑、伐木等等，每种行为可以对应一小段脚本
- 搜索算法可以用来做决策，例如决定何时进行某种行为
- 本文将决策树和树搜索算法来进行决策，输入游戏状态，输出下一步的行为

### 前向搜索
- 前向搜索一般有两种实现，第一种是复制一份游戏状态，在新状态上进行搜索；另一种是直接进行搜索，结束后进行回退
- 回退方法一般用于卡牌等状态比较简单的方法，复制方法一般用于RTS等状态比较多的游戏
- 决策树不仅要考虑自己，也要考虑到对手行为，一般可以用Minimax搜索来完成决策，对于RTS这样的零和游戏，可以用NegaMax
```python
def negaMax(state, depth, player):
    if depth == 0 or terminal(state):
        return evaluate(state, player)
    max = -float('inf')
    for move in state.legal_moves(player):
        childState = state.apply(move)
        score = -negaMax(childState, depth-1, opponent(player))
        if score > max:
            max = score
    return max
#example call
#state: current game state
#depth: maximum search depth
#player: player to move
value = negaMax(state, depth, player)
```
- 由于RTS游戏是实时的，不是轮流的回合制，因此NegaMax算法实现需要改变
```python
def SMNegaMax(state, depth, previousMove=None):
    player = playerToMove(depth)
    if depth == 0 or terminal(state):
        return evaluate(state, player)
    max = -float('inf')
    for move in state.legal_moves(player):
        if previousMove == None:
            score = -SMNegaMax(state, depth-1, move)
        else
            childState = state.apply(previousMove, move)
            score = -SMNegaMax(childState, depth-1)
        if score > max:
            max = score
    return max
#Example call
#state: current game state
#depth: maximum search depth, has to be even
value = SMNegaMax(state, depth)
```
- MCTS也可以应用在上述情况

### 应用
- 游戏决策必须和行为执行并行
- 星际争霸：母巢之战的AI应用了上述方法，不过没有考虑到战争迷雾，这就需要AI去猜想敌人的当前行为
