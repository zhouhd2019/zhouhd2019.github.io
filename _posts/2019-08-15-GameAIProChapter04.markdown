---
layout:     post
title:      "GameAIPro Chapter04"
subtitle:   "Behavior Selection Algorithms"
date:       2019-08-15 16:45:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameProgramming
---

有限状态机 Finite State Machines
-----------------
```c++
class FSMState
{
    virtual void onEnter();
    virtual void onUpdate();
    virtual void onExit();
    list<FSMTransition> transitions;
};

class FSMTransition
{
    virtual void isValid();
    virtual FSMState* getNextState();
    virtual void onTransition();
};

class FiniteStateMachine
{
    void update();
    list<FSMState> states;
    FSMState* initialState;
    FSMState* activeState;
};
```
   - 经典AI实现方式，将AI分为多个状态，一般一个时刻只会处于一个状态，状态间可以发生转变
   - 对于多个类似的状态，可以将它们整合为一个大的状态，这就是Hierarchical Finite State Machine分层有限状态机，这样可以有效减少状态间转移的数量。一般大的状态还会带一个特别的指针history指向上次运行的子状态，这样再次进入时可以从上次的状态继续运行

行为树 Behavior Trees
-----------------
   - 将AI行为用树进行组织，每个节点拥有条件，符合条件才进入子树，直到找出可执行节点
   - 和FSM相比，BT基本是无状态的，不需要考虑节点间转移，实现简单，容易扩展，但由于很多时候需要从头判断该执行哪个节点，消耗要更大

效用系统 Utility System
-----------------
   - 一般的AI决策都过于绝对，效用系统是对多个行为进行评分，例如根据子弹数量和自身血量来决定撤退与否
   - 根据评分进行选择可以使用不同的策略，例如选择评分最高的，或者根据评分来随机选择
   - 效用系统的随机性比其它方法要大，debug和调试更难

目标导向行动规划 Goal Oriented Action Planners
-----------------
   - GOAP会给角色多个目标和这些目标的完成前提条件，AI会规划出达到所需目标的行动序列
   - GOAP常用规划方法是后向链式搜索，从目标反推可行序列。不过，相对复杂的前向链式搜索，用到启发式搜索和剪枝等技巧，效率会更高

分层任务网络 Hierarchical Task Networks
-----------------
   - HTN和GOAP一样，也是行为规划的一种，不过规划的具体方法不一样，HTN从一个大的根任务出发，持续将它分解为多个子任务，直到子任务可执行
   - GOAP是寻找行动序列来完成任务，HTN是将任务分解为子任务
