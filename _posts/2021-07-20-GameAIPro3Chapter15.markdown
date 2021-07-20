---
layout:     post
title:      "Steering against Complex Vehicles in Assassin's Creed Syndicate"
subtitle:   "GameAIPro3 Chapter15"
date:       2021-07-20  11:50:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro3
    - GameProgramming
---

本章讲述了刺客信条枭雄如何避让复杂车辆，避让计算的方法是简单的推力法，主要的改动在于采用凸包来描述复杂角色

## 现状
- 常见避让方法会将角色简化成圆形或者椭圆，枭雄里面常见的马车很难进行上述简化
- 当马车停下来，逻辑会将马车占据的部分设置为障碍，分帧重算马车所在寻路网格，更新后就不需要进行避让
- 当快要碰撞时，避让系统会给角色一个侧向推力来改变角色行走方向
- 如果用椭圆分别简化马和车，角色容易卡在它们之间

## 凸包生成
- 马车最好的简化体是凸包，它能够将马车视作一个整体，不会创建出任何凹形部分
- 凸包方法还可以兼容其它情况，例如没有马的车
- 要生成凸包直接用现有方法即可，下面是刺客信条用到的方法，比较繁琐
- 首先需要将刚体简化为2D的OOBB，多个刚体的角色就有多个OOBB
- 接下来找出OOBB的轮廓点，例如马车可能两个刚体存在部分重合，需要计算出交点
- 对于两个相交的OOBB，一种可行的算法是先找到一个只在一个OOBB上的顶点(不在其它OOBB内)，顺时针遍历该OOBB的其它顶点，遇到相交的边就计算交点，并遍历相交的OOBB，以此类推直到遍历完初始OOBB的所有顶点
```c++
currVtx = newShapeToAdd;
altVtx = existingContour;

firstOutsideIndex = firstOutsideVtx(currVtx, altVtx);
nextVtx = currVtx[firstOutsideIndex];
nextIdx = circularIncr(firstOutsideIndex, 1, currVtx.size());

mergedContour.push(nextVtx);
while(!looped)
{
    intersections = collectIntersections(nextVtx, currVtx[nextIdx], altVtx);
    if(intersections.empty())
    {
        nextVtx = currVtx[nextIdx];
        nextIdx = circularIncr(nextIdx, 1, currVtx.size());
    }
    else
    {
        intersectionIdx = findClosest(nextVtx, intersections);
        nextVtx = intersections[intersectionIdx];
        // since we’re clockwise, the intersection can store
        // the next vertex id
        nextIdx = intersections[intersectionIdx].endIdx;
        swap(currVtx, altVtx);
    }
    if(mergedContour[0].equalsWithEpsilon(nextVtx))
        looped = true;
    else
        mergedContour.push(nextVtx);
}
```
- 结果得到的顶点数如果和初始OOBB的一样，说明另一个OOBB在初始OOBB内或者完全没有相交
- 对于多于两个OOBB的情况，可以先处理两个，再逐个添加

## 障碍避让
- 对于要绕过马车的角色，可以让他和马车凸包进行碰撞检测，并在碰撞点往外加一个推力
- 对于要走近马车的角色，需要让他和马车轮廓进行碰撞检测
- 对于要走出马车的角色，需要先和轮廓进行检测，当走出凸包后再和凸包进行碰撞检测
