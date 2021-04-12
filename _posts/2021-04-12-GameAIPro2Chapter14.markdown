---
layout:     post
title:      "JPS+: An Extreme A* Speed Optimization for Static Uniform Cost Grids"
subtitle:   "GameAIPro2 Chapter14"
date:       2021-04-12  16:10:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameAIPro2
    - GameProgramming
---


## JPS和JPS+
- 和传统A*方法相比，JPS方法快一个量级以上。如果加上离线静态地图分析，可以再快一个量级，这种方法叫做JPS+
- JPS有两个特点，一个是不会重复遍历格子，每个格子只会有一种路径可以到达，简单来说就是不会折返；另一个特点就是只有部分点会进入open list，JPS将它们命名为跳点
- 只有正向移动才会遇到强迫邻居，表示接下来的搜索需要考虑更多方向的节点，否则会错过一部分
- 跳点有四种：基本跳点，拥有强迫邻居的点，注意和移动方向有关；直线跳点，通过正向移动可以到达基本跳点的点；对角线跳点，通过对角线移动可以到达基本跳点或者直线跳点的点；起点和终点

## JPS+预处理
- 对每个可行走格子进行预处理，计算到跳点距离用正数表示，如果该方向没有跳点，就计算离墙壁的距离，用负数表示
- 具体流程：
    1. 找出所有基本跳点，记录跳点方向
    2. 从左到右遍历地图，找出所有直线跳点，记录每个点到左边的直线跳点或者墙壁距离
    3. 和步骤2同理，从右到左、从上到下、从下到上找出直线跳点，并记录每个点到直线跳点或者墙壁距离
    4. 从下到上，找出对角线跳点，并计算左下和右下方向的跳点距离或墙壁距离
    5. 和步骤4同理，从上到下，处理左上和右上方向
- 计算左向距离的伪代码，其它正向距离类似
```c++
for (int r = 0; r < mapHeight; ++r>) {
    int count = -1;
    bool jumpPointLastSeen = false;
    for (int c = 0; c < mapWidth; ++c) {
        if (m_terrain[r][c] == TILE_WALL) {
            count = -1;
            jumpPointLastSeen = false;
            distance[r][c][West] = 0;
            continue;
        }
        count++;
        if (jumpPointLastSeen) {
            distance[r][c][West] = count;
        } else {
            distance[r][c][West] = -count;
        }
        if (jumpPoints[r][c][West]) {
            count = 0;
            jumpPointLastSeen = true;
        }
    }
}
```
- 从下往上计算所有左下对角线跳点或者墙壁距离，其它方向同理
```c++
for (int r = 0; r < mapHeight; ++r) {
    for (int c = 0; c < mapWidth; ++c) {
        if (!IsWall(r, c)) {
            if (r == 0 || c == 0 || IsWall(r - 1, c) || IsWall(r, c - 1) || IsWall(r - 1, c - 1)) {
                distance[r][c][Southwest] = 0;  // 旁边就是墙壁
            }
            else if (!IsWall(r - 1, c) && !IsWall(r, c - 1) && 
                (distance[r - 1][c - 1][South] > 0 || distance[r - 1][c - 1][West] > 0)) {
                distance[r][c][Southwest] = 1;  // 旁边是直线跳点
            } 
            else {
                int jumpDistance = distance[r - 1][c - 1][Southwest];
                if (jumpDistance > 0) {
                    distance[r][c][Southwest] = 1 + jumpDistance;
                } else {
                    distance[r][c][Southwest] = -1 + jumpDistance;
                }
            }
        }
    }
}
```
- JPS+运行时伪代码
```c++
/*
ValidDirLookUpTable
    Traveling South: West, Southwest, South, Southeast, East
    Traveling Southeast: South, Southeast, East
    Traveling East: South, Southeast, East, Northeast, North
    Traveling Northeast: East, Northeast, North
    Traveling North: East, Northeast, North, Northwest, West
    Traveling Northwest: North, Northwest, West
    Traveling West: North, Northwest, West, Southwest, South
    Traveling Southwest: West, Southwest, South
*/
while (!OpenList.IsEmpty()) {
    Node* curNode = OpenList.Pop();
    Node* parentNode = curNode->parent;
    if (curNode == goalNode)
        return PathFound;
    
    foreach (direction in ValidDirLookUpTable given parentNode) {
        Node * newSuccessor = NULL;
        float givenCost;

        if (direction is cardinal && goal is in exact direction && 
            DiffNodes(curNode, goalNode) <= abs(curNode->distances[direction])) {
            newSuccessor = goalNode;
            givenCost = curNode->givenCost + DiffNodes(curNode, goalNode);
        } else if (direction is diagonal && goal is in general direction && 
            (DiffNodesRow(curNode, goalNode) <= abs(curNode->distances[direction]) || 
            (DiffNodesCol(curNode, goalNode) <= abs(curNode->distances[direction])))) {
            int minDiff = min(RowDiff(curNode, goalNode), ColDiff(curNode, goalNode));
            newSuccessor = GetNode(curNode, minDiff, direction);
            givenCost = curNode->givenCost + (SQRT2 * minDiff);
        } else if (curNode->distances[direction] > 0) {
            newSuccessor = GetNode(curNode, direction);
            givenCost = DiffNodes(curNode, newSuccessor);
            if (diagonal direction) {givenCost *= SQRT2;}
            givenCost += curNode->givenCost;
        }

        if (newSuccessor != NULL) {
            if (newSuccessor not on OpenList or ClosedList) {
                newSuccessor->parent = curNode;
                newSuccessor->givenCost = givenCost;
                newSuccessor->finalCost = givenCost + CalculateHeuristic(newSuccessor, goalNode);
                OpenList.Push(newSuccessor);
            } else if (givenCost < newSuccessor->givenCost) {
                newSuccessor->parent = curNode;
                newSuccessor->givenCost = givenCost;
                newSuccessor->finalCost = givenCost + CalculateHeuristic(newSuccessor, goalNode);
                OpenList.Update(newSuccessor);
            }
        }
    }
}
```
- JPS+对原来的JPS方法主要改进的地方在于预先知道需要遍历的方向和距离，不需要逐个方向和格子遍历，但要求地图不能动态修改
