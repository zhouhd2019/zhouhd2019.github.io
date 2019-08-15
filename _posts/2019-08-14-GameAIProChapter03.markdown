---
layout:     post
title:      "GameAIPro Chapter03"
subtitle:   "Advanced Randomness Techniques for Game AI"
date:       2019-08-15 11:13:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameProgramming
---

正态分布
-----------------
   - 扔多次骰子，所得数之和会逐渐接近正态分布。自然界里有很多事情和正态分布相关，原因和上述类似，多个因素的共同影响导致分布类似正态分布。这就是中心极限定理
   - 根据中心极限定理，K个均匀分布之和，标准差为$$\sqrt{K/3}$$，所以正态分布可以用3个来简单模拟，代码如下：
```c++
unsigned long seed = 61829450;
double GaussianRand()
{
    double sum = 0;
    for(int i = 0; i < 3; i++)
    {
        unsigned long holdseed = seed;
        seed ^= seed << 13;
        seed ^= seed >> 17;
        seed ^= seed << 5;
        long r = (Int64)(holdseed + seed);
        sum + = (double)r * (1.0/0x7FFFFFFFFFFFFFFF);
    }
    return sum; //returns [-3.0, 3.0] at (66.7%, 95.8%, 100%)
}
```
   - 正态分布可以用在很多地方，例如射击游戏中的子弹分布，在极坐标上分别对角度和距离进行正态分布随机

过滤随机数
-----------------
   - 简单地用随机函数得到的结果有时不太好，会给玩家带来不好的体验，需要过滤一下，主要有：多个连续相同数字，模式重复例如11001100，某个数字在一定区间内出现过多，连续等差数字
   - 如果出现不好的结果，可以考虑再随机一次，替换刚才的结果，还可以先随机完毕，再进行过滤
   - 对于一个随机数序列，可以用ENT来评价它的随机性

柏林噪声
-----------------
   - 一维柏林噪声是连续的，非常有用的特性，用在移动和动画等方面可以减少突变行为
   - 柏林噪声分为两种，Simplex噪声和分形噪声
   - 一维Simplex噪声在x轴每个整数坐标上随机生成1个数作为梯度，然后用一个插值函数对两点之间进行平滑。二维可以在正方形中插值，也可以在等边三角形中插值
   - 分形噪声就是不断叠加更高频率的柏林噪声的结果
