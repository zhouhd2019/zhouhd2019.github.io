---
layout:     post
title:      "Procedual Level and Story Generation Using Tag-Based Content Selection"
subtitle:   "GameAIPro3 Chapter38"
date:       2021-08-10  20:58:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

基于标签的资源选择，一种比较简单的方法，可用于程序化关卡和故事生成

## 概述
- 对每个资源进行标记，也就是用一个或者多个字符串来描述资源的特点，使用时给出一个或者多个标签，根据这些标签找到符合要求的资源，这就是基于标签的资源选择方法
- 一种常见问题是没有找到符合要求的资源，遇到这种情况说明资源不足或者标签的组合有问题，可以做一个检查工具，持续地验证现有资源不会出现不能满足要求的情况
- 如果用于生成关卡或故事的某种资源比较少，容易出现重复感，最好保证这个问题不会很严重，简单增加资源即可
- 可选标签，是指如果没有符合要求的资源，可以忽略的标签，这是个不错的功能
- 排除标签，是指目标资源不要带有这种标签。实现上可以简单地先选出符合其它标签的资源，再进行过滤排除。如果排除标签种类较少，也可以提前先分好类别
- 如果选择多个相同条件的资源，要考虑是一次抽取还是分多次抽取
