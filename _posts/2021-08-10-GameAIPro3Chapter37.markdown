---
layout:     post
title:      "Simulating Character Knowledge Phenomena in Talk of the Town"
subtitle:   "GameAIPro3 Chapter37"
date:       2021-08-10  15:00:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

本章讲的是游戏Talk of the Town的一个核心系统实现，让NPC拥有认知能力，可以和其他角色交流信息，信息还可以随着时间发生变化

## 概述
- 很少游戏会对NPC的记忆和认知进行建模，一般都是简单的手写功能
- Talk of the Town的玩法是不对称对抗，一个玩家伪装成NPC，其他玩家要找出伪装玩家，NPC和可以和玩家对话，提供或者获得一些信息。伪装玩家需要向NPC散布谎言，混淆NPC对自身的认知。其他玩家需要获得尽可能多的信息，并分辨出信息的真假，从而在最后的聚会中找到伪装玩家
- Talk of the Town的核心系统之一是NPC认知系统，NPC可以从周围环境获得信息，也可以和其他角色交流信息，并且认知会随着时间发生变化

## 模拟需求
- 认知信息生命周期主要有4部分，产生、传播、变化和消失
- 角色有很多属性，其他角色可以直接观察得到，例如头发颜色和工作地点
- 当一个角色和其他角色距离不远，他可以观察到其他角色，如果很近，还可以听到其他角色说的话
- 每个角色的每个属性都有自己的显著程度，越显著的属性越先被其他角色观察并记住
- 认知记忆可能会发生变化，游戏需要考虑到每种属性的合理变化可能

## 信息
- 一个角色的认知信息应该可以组织成一个网络，例如一个人工作在某个地方，那么这个人的工作属性就应该指向所在公司，这个公司的雇员属性会指向多个角色
- 每条信息都有自身属性：值，例如发色属性的值可能是黑色；所属模型；信息来源；证据和可信度等等
- 信息来源：自省，不需要模拟，每个角色能知道自己所有信息；观测，直接看到别的角色，显著程度越高的信息越先被记住；转移，如果角色发现两个别的角色比较相似，会将他们的一些信息互相复制；虚构，无意识的捏造信息；撒谎；植入，某些基础信息会直接设置
- 信息加强：一个角色描述一个信息越多，他就越相信这个信息
- 信息变化：记忆出错
- 信息消失：被遗忘
- 信息的证据本身也有一些属性：证据来源，例如别人描述还是自己偷听；获得证据的地方、时间；证据强度，随着时间逐渐降低

## 实现细节
- 显著程度，决定信息是否会被记住，是否会被记错，是否会被遗忘。具体计算和很多因素相关，例如观察者和被观察者的关系，观察者对该种信息的感兴趣程度，事情发生位置
- 显著程度还决定信息被传播的概率，当两人交流，所聊的信息从最显著的几个信息中选择
- 属性信息被记错的结果和属性本身相关，例如黑色头发会被记错为其它深色头发的可能性会更大
- 如果某个信息遇到了可信度更高的反面证据，需要对信息做出修正
- 游戏开始时预先设定一些信息，例如家人和朋友关系，伪代码如下：
```
for resident of town
    implants = []
    for immediate family member of resident
        add immediate family member to implants
    for friend of resident
        add friend to implants
    for neighbor of resident
        add neighbor to implants
    for coworker of resident
        add coworker to implants
    for every other character who has ever lived
        chance = 1.0 - (1.0/salience of that character)
        if random number < chance
            add other character to implants
    for character in implants
        for attribute of character
            chance = attribute salience
            chance += -1.0/salience of that character
            if random number < chance
                have resident adopt accurate belief for attribute
```
- 整个游戏的核心流程伪代码如下
```
do world generation // See Section 37.2.2
do belief implantation // See Listing 1
while central character still alive // One week of game time
    advance one timestep
    for resident in town
        enact resident routine // Put somewhere in the town
    for resident in town
        for nearby character at same place as resident
            if characters will interact // See Section 37.2.2
                do knowledge exchange // See Section 37.3.7
    for resident in town
        do simulation of fallibility phenomena // See Section 37.3.8
```
