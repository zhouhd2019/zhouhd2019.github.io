---
layout:     post
title:      "Monte Carlo Tree Search and Related Algorithms for Games"
subtitle:   "GameAIPro2 Chapter25"
date:       2021-07-05  11:19:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro2
    - GameProgramming
---

如题，蒙特卡洛树搜索和相关算法的简介

## 背景
- 在线算法是指玩家游玩游戏同时运行的算法，离线算法是指预先训练好的策略
- 本文主要解决多臂赌博机这类问题，面对多个老虎机，如何玩才能获得最大收益，下面会用最简单的剪刀石头布为例描述算法，即三臂赌博机
- 本文用效用这个词来代表某个行为的可能收益，效用越高，可能行为收益越高

## 算法1：Online UCB1
- 公式如下，v(i)是第i个赌博机臂的效用，x(i)是过去的平均效用，c(i)是该机臂的使用次数，t是总次数，k用来平衡探索exploration和获益exploitation的常量
```
v(i) = x(i) + sqrt(k * ln(t) / c(i))
```
- 探索是指选择没有进行过的行为，获益是指选择当前效用最高的行为
- x(i)是获益相关计算，另一部分v(i)公式是探索计算，选择次数c(i)多了就会变小
- Online UCB1适用于选择多种有意义的行为
- 如果某种怪物有多套AI，可以使用上述方法来获得更适合当前玩家的AI
- 不适合选择武器的场合，怪物会花不少时间来尝试比较差的武器

## 算法2：Regret Matching
- 在线学习或者离线训练都可以，每一轮选择后，对于没有选择的其它行为，累计它们的后悔值。越有效的行为，后悔值会越高
- 刚开始时，每个行为后悔值都是统一的初始值，随机选择。后面就根据后悔值，对后悔值不为负的行为进行加权随机。执行行为后，根据本次情况，计算其它行为的效用，并将它处理，例如最简单的减去本次行为的效用，所得结果累计到各自行为的后悔值
- 如果一种行为能够给出更好的收益，它的后悔值会越来越高，从而在随机选择中更频繁出现
- Regret Matching不能用于那些很难直接计算效用的场景

## 算法3：Offline UCB1
- 离线训练时，如果可选行为很多，会使得分支很多，模拟时间过长，可以通过UCB1来减少低收益分支的模拟
- 伪代码如下：
```javascript
function SimulateUCB()
{
    while (time remains)
    {
        act = GetNextAction();  // UCB1选择一种行为
        ApplyAction(act);
        utility = PlayDefaultStrategy();
        UndoAction(act);
        TellUtility(act, utility);
    }
    return GetBestAction();
}

function TellUtility(act, utility)
{
    totalActions++;
    score[act] += utility;
    count[act]++;
}
```

## 算法4：UCT
- Offline UCB1是一种一层搜索算法，每种行为之间都是平等独立的，但实际应用中很多行为存在一些关联
- 将Offline UCB1扩展为多层结构，即树型结构，可以很好考虑到关联行为，UCB Tree被称为UCT，是蒙特卡洛树搜索MCTS的一种
- 一次训练过程分为四步：选择，根据UCB1选择本次训练的分支；扩展，在选择的分支下增加新的行为；模拟，使用默认策略对该分支进行模拟；反向传播，根据最终模拟结果，更新本节点到根节点整条分支路径的节点的效用值
- 伪代码如下：
```javascript
function SimulateUCT()
{
    while (time remains)
    {
        TreeSelectionAndUpdate(root, false);
    }
    return GetBestAction();
}

function TreeSelectionAndUpdate(currNode, simulateNow)
{
    if (GameOver(currNode))
        return GetUtility(currNode);
    if (simulateNow)
    {
        //Simulate the rest of the game and get the utility
        value = DoPlayout(currNode);
    }
    else if (IsLeaf(currNode))
    {
        AddChildrenToTree(currNode);
        value = TreeSelectionAndUpdate(currNode, true);
    }
    else {
        child = GetNextState();//using UCB1 rule (in tree)
        value = TreeSelectionAndUpdate(child, false);
    }
    //If we have 2 players, we would negate this value if
    //the second player is moving at this node
    currNode.value += value;
    currNode.count++;
    return value;
}
```
- 对于游戏模拟，禁止加血等回溯类行为可以有效减少模拟时间
