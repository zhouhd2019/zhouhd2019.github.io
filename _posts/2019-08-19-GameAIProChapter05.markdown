---
layout:     post
title:      "GameAIPro Chapter05"
subtitle:   "Structual Architecture Common Tricks of the Trade"
date:       2019-08-19 17:55:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameProgramming
---

## 选项栈 Option Stacks
   - 一般AI实现难以处理突发情况，例如正在攻击敌人，一颗手雷落到脚边，应该先躲避，躲避完毕再回来继续攻击敌人
   - 选项栈就是一个简单的栈，AI的每个决策都会依次入栈，处理完毕就出栈，这样就可以很简单处理突发情况，并且再处理完毕后回到之前的行动决策
   - 选项栈还可以处理角色状态相关逻辑，例如被击中要播放受击动画，可以将受击放到选项栈

## 知识管理 Knowledge Management
   - 实际上就是说一些AI用到的信息要怎么存取，最简单的方法就是blackboard，可以简单实现为一个dict，blackboard一般放到character AI代码中
   - AI相关信息不只可以放在character，还可以放到游戏世界中，放在可交互物体/资源点，例如在模拟人生，电视存了看电视的角色应该播放的动作和当前看电视人数

## 模块性 Modularity
   - 通过模块化AI可以减少重复代码和计算，AI决策通常要考虑很多因素，例如逃跑需要考虑敌人位置/状态，本人状态等等，这些信息可以统一放一起给AI使用
   - 寻找目标是一个应该独立的AI小模块，效用系统里的评分函数（给每个因素打分）也应该独立出来，减少重复计算
