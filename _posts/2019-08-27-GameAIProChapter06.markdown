---
layout:     post
title:      "GameAIPro Chapter06"
subtitle:   "The Behavior Tree Starter Kit"
date:       2019-08-27 17:20:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameProgramming
---

## 行为树基本组成
   - 行为基类，主要有三个接口，初始化/每帧更新/退出运行。下面是示例代码，注意每次检查是不是正在运行来决定执行初始化或者退出运行函数，这样非常不好，应该放到行为树中，而不是节点，这里只是示例

```
class Behavior {
protected:
    virtual void onInitialize() {}
    virtual Status update() = 0;
    virtual void onTerminate(Status) {}
private:
    Status m_eStatus;
public:
    Behavior() : m_eStatus(BH_INVALID) {}
    virtual ~Behavior() {}
    Status tick() {
        if (m_eStatus != BH_RUNNING) onInitialize();
        m_eStatus = update();
        if (m_eStatus != BH_RUNNING) onTerminate(m_eStatus);
        return m_eStatus;
    }
}
```

   - 节点更新时应该返回状态，一般是成功/失败/运行/暂停/错误/异常等等，行为树根据状态来决定下一步
   - 行为Actions，执行具体的行为逻辑
   - 条件Conditions，一般有两种，即时检查和监控
   - 装饰器Decorators，用来给行为添加细节，修改返回结果等等，例如重复执行多次某个行为节点
   - 复合节点Composites，拥有至少一个子节点，并决定该怎么运行子节点，例如顺序执行Sequences/Selectors选择节点/平行节点Parallels
   - 过滤节点Filters，顺序节点的一种，如果条件不满足就不会继续执行
   - 监控节点Monitors，用在平行节点，持续检查条件是否满足
   - 主动选择节点Active Selectors，就是Unreal里面的打断机制，优先级高的节点可以打断低优先级节点的运行，这种打断应该支持事件和每帧检测两种

## 行为树实现
   - 行为树可以分为两代，第一代行为树比较矮小，复杂节点用C++实现，多个行为树之间基本没有共享，实现比较简单不需要担心性能，通常用C++实现
   - 第二代行为树一般比较大比较复杂，每个节点更小更容易组合使用，数据可以在多个行为树之间共享，需要高度优化，可以重用于其它逻辑，一般用DSL实现
   - 并不是说第二代一定比第一代好，第一代更简单更好理解
   - 多个实体间共享行为树可以大大减少内存消耗，例如共享同一个行为树或者子树，可以节省节点间的指针等等，这样就需要将各自的运行时数据抽出来
   - 提前统一分配行为树节点内存可以有效减少卡顿，并且让同一棵树的节点能够相邻分布，减少cache miss。还有复合节点，可以将子节点直接存放在数组
   - 行为树的执行尽量减少每帧update，多使用事件，确定的行为直接编码，例如顺序节点运行完一个子节点就执行下一个
