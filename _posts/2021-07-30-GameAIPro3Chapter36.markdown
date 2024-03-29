---
layout:     post
title:      "Stochastic Grammars: Not Just for Words"
subtitle:   "GameAIPro3 Chapter36"
date:       2021-07-30  18:40:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

本章讲的是如何在游戏中应用形式文法和EBNF，可以给AI的随机行为加上一定的规则，可以用于简单AI的实现，可以生成一些简单的脚本

## 概述
- 形式文法广泛用于字符串和规则定义，其中最广泛使用的描述方法是EBNF
- EBNF也可以用来描述角色AI行为。例如BOSS拥有三个技能，提升敌人着火概率的弱火术、火球术、重伤着火敌人的终结技。一般技能循环是弱火术、火球术最后是终结技，但这样过于固定，应该加入一些随机性，例如
```
弱火术连击--------弱火术，{火球术}
基本连招----------火球术，{弱火术连击}，终结技
攻击连招----------{火球术}，{基本连招}
```

## 随机形式文法
- 除了常见的状态机，还可以将简单的EBNF实现为树型结构，每条规则是树的一个节点，嵌套的子规则是父规则的子节点。对于有多种可能的节点，生成时会根据概率返回一个子节点的结果
- 对于需要生成无限长度的情况，一般采用流式方法进行处理
- 形式文法很多时候和行为树类似，一个关注行为序列，而行为树关注不同情况的应对行为。相比之下，形式文法在实现上更简单，不过功能也没行为树强大
- 形式文法可以用来生成一些简单的命令甚至脚本
- 对于有多种可能的随机规则，如何给出合理的概率是一个很重要的问题，可以通过机器学习和效用AI等方法来获取合理数值
