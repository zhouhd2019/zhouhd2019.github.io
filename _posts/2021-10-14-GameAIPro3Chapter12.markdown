---
layout:     post
title:      "A Reusable, Light-Weight Finite-State Machine"
subtitle:   "GameAIPro3 Chapter12"
date:       2021-10-14  14:43:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

本篇经验来自Drawn to Life: The Next Chapter，讲如何实现一个简单的状态机。

### 基础实现
- 状态机逻辑由三部分组成，状态、状态转移和状态机
- 状态实现主要包括进入、退出和更新
```c++
class State
{
    GameObject* m_pOwner;
public:
    State(GameObject* pOwner) : m_pOwner(pOwner) {}
    virtual void OnEnter() {}
    virtual void OnExit() {}
    virtual void OnUpdate(float deltaTime) {}
protected:
    GameObject* GetOwner() const { return m_pOwner; }
}
```
- 状态机需要保存状态和状态之间的转移，还有当前状态
```c++
class StateMachine
{
    typedef pair<Transition*, State*> TransitionStatePair;
    typedef vector<TransitionStatePair> Transitions;
    typedef map<State*, Transitions> TransitionMap;

    TransitionMap m_transitions;
    State* m_pCurrState;
public:
    void Update(float deltaTime);
};

void StateMachine::Update(float deltaTime)
{
    // find the set of transitions for the current state
    auto it = m_transitions.find(m_pCurrState)
    if (it != m_transitions.end())
    {
        // loop through every transition for this state
        for (TransitionStatePair& transPair : it->second)
        {
            // check for transition
            if (transPair.first->ToTransition())
            {
                SetState(transPair.second);
                break;
            }
        }
    }

    // update the current state
    if (m_pCurrState)
        m_pCurrState->Update(deltaTime);
}
```
- 状态转移只需要实现检查当前能否转移
```c++
class Transition
{
    GameObject* m_pOwner;
public:
    Transition(GameObject* pOwner) : m_pOwner(pOwner) {}
    virtual bool ToTransition() const = 0;
}
```

### 进阶
- 可以支持分层状态机，每个状态本身也可以是一个状态机，内部包含子状态
- 状态应该尽量抽象，例如播放不同动画的行为都可以用播动画这一个状态
- 减少更新频率，每帧只更新部分对象的状态机，可以降低更新压力
- 将状态定期Update改为事件驱动，可以减少无用计算
- 将数据和代码分离，例如放到Blackboard，可以减少内存
