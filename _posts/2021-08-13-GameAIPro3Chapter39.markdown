---
layout:     post
title:      "Recommendation Systems in Games"
subtitle:   "GameAIPro3 Chapter39"
date:       2021-08-13  18:58:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

推荐系统入门，不过游戏里面用得不是很多

## 基于内容
- 预先对物品进行标记，推荐时根据用户已有物品的标记，来推荐拥有相似标记的物品
- 举个例子，玩家买了一些FPS，系统会给他推荐FPS
- 当用户没有已购买物品时，这个算法不能运行，有冷启动问题。另外需要好好维护物品的标记

## 协同过滤推荐
- 由其他用户的购买数据，来推断当前用户可能感兴趣的物品
- 不需要对物品进行预先标记
- 可以用于物品评分和物品排名
- 基于用户的算法，根据拥有类似物品的用户，给出推荐，伪代码如下
```
For every other user V
    Compute the similarity S between U and V
    For every item I rated by V
        Add V's rating for I weighted by S to I's avg.weight
Return the top rated items
```
- 基于物品的算法，根据当前用户已有物品，推荐类似物品
```
For every item I
    For every item J already rated by U
        Compute the similarity S between I and J
        Add U's rating for J weighted by S to I's avg.weight
Return the top rated items
```
- 基于物品的算法，物品相似度可以通过用户同时拥有两者的情况类决定

## 基于模型过滤推荐
- 预测性模型可用于推荐系统，例如推算用户对物品的感兴趣程度
- 需要大量数据和离线训练

## 算法选择
- 需要考虑很多因素，例如系统是否需要评分和排名，物品数量，有没有合适的标记数据，用户数量等等