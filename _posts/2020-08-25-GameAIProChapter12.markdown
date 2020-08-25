---
layout:     post
title:      "Exploring HTN Planners through Example"
subtitle:   "GameAIPro Chapter12"
date:       2020-08-25 15:56:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameProgramming
---

## HTN
- Hierarchical Task Networks(HTN)将一个task作为输入，返回一系列步骤来完成task。这一些列步骤被称为plan。求解过程中，HTN递归地将task分解为更小的task，直到不可再分
- World State: 每个agent会用或者类似概念来保存它关心的状态，例如当前HP/敌人位置等等
- Sensors: 监测关心的游戏状态，并将游戏里面的状态转换为agent可以理解的状态，保存到World State
- Primitive Task: 原始任务是指agent可以完成而且不可再分的任务，用于组成复合任务。定义一个原始任务需要条件/行为/结果，必须是清晰且可描述的
- Compound Task: 可以由Primitive Task组合而成的任务，一般有多种组合方法
- Domain: 整个任务架构
- 每次求解循环，planner会将一个task从TaskToProcess栈弹出，如果是Compound Task则进行分解，如果是Primitive Task则检查前置条件
```
WorkingWs = CurrentWorldState
TasksToProcess.Push(RootTask)
while TasksToProcess.NotEmpty {
    CurrentTask = TasksToProcess.Pop()
    if CurrentTask.Type == CompoundTask {
        SatisfiedMethod = CurrentTask.FindSatisfiedMethod(WorkingWS)
        if SatisfiedMethod != null {
            RecordDecompositionOfTask(CurrentTask, FInalPlan, DecompHistory)
            TasksToProcess.InsertTop(SatisfiedMethod.SubTasks)
        } else {
            RestoreToLastDecomposedTask()
        }
    } else { // Primitive Task
        if PrimitiveConditionMet(CurrentTask) {
            WorkingWS.ApplyEffects(CurrentTask.Effects)
            FinalPlan.PushBack(CurrentTask)
        } else {
            RestoreToLastDecomposedTask()
        }
    }

}
```

## 实践细节
- 要实现重新规划功能，即优先级更高的任务可以打断当前任务，需要在某些时机(例如World State变化/任务开始前)检查更高优先级的任务能否执行。同时需要注意避免一些重要任务被意外打断
- 有时需要同时两个任务一起执行，例如举起盾牌并前进。实现方法有几种：实现一个能完成两种任务的状态，封装两种任务，例如实现一个举盾前进的技能；使用两个planer；将其中一个任务的结果实现为状
- 为了避免搜索时间过长，应该支持部分搜索，但这样会变得非常复杂
