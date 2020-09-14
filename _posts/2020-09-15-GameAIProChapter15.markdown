---
layout:     post
title:      "Runtime Compiled C++ for Rapid AI Development"
subtitle:   "GameAIPro Chapter15"
date:       2020-09-15  23:38:00
author:     "zhouhd"
header-img: "img/about-bg.jpg"
catalog: true
tags:
    - GameAI
    - GameAIPro
    - GameProgramming
---

## 常见方法
- 脚本语言：整合开销/性能/工具和DEBUG/底层支持如数学运算等等方面，不能让人满意。部分游戏和平台不允许热更和JIT，用不上脚本的相关功能
- Multiple Processes，简单来说就是分离数据和代码，将数据放在共享内存等地方，需要改变代码时，就只需要保存一下数据，更新代码，再让新代码载入数据。消耗可能还是有些大
- Data Driven Approaches，数据配置（例如用XML）和代码分开，灵活性不足，或者配置过多
- 可视化脚本，基本是更好的DDA
- Library Reloading，DLL重载，实际使用中很难平衡DLL大小和数量，过大会导致编译和重载时间很长，过小会导致DLL过多

## Runtime Compiled C++
- 和UE4的Hot Reload类似，支持实时C++代码reload
- 两部分，实时编译器和实时对象系统。前者处理文件改变通知和编译。后者跟踪源文件和它们的依赖关系，提供IObject基类接口和IObjectFactorySystem，当新代码载入时创建对象并替换旧对象
- 代码实时编译失败，需要捕捉异常并提供相关信息进行DEBUG。另一个可选的高级特性是提供回退，不过其实可以通过简单回退代码重新编译来实现
- 序列化和反序列化主要通过变量名字来对应。另外头文件依赖追溯可以通过宏来实现
- 函数实时替换通过virtual增加一个中间层来实现，宏定义virtual关键字可以让release版本去掉不需要的运行时编译

## 具体实现: ConsoleExample
- 入口ConsoleExample.cpp的main，具体逻辑只有如下几行
```c++
ConsoleGame game;
if(game.Init()) {
    while(game.MainLoop()) {}
}
```
- ConsoleGame继承于IObjectFactoryListener，后者只是一个接口类。ConsoleGame::Init函数创建了RuntimeObjectSystem和RuntimeObject01实例
```c++
bool ConsoleGame::Init() {
	//Initialise the RuntimeObjectSystem
	m_pRuntimeObjectSystem = new RuntimeObjectSystem;
	m_pRuntimeObjectSystem->Initialise(m_pCompilerLogger, 0);
	m_pRuntimeObjectSystem->GetObjectFactorySystem()->AddListener(this);

	// construct first object
	IObjectConstructor* pCtor = m_pRuntimeObjectSystem->GetObjectFactorySystem()->GetConstructor( "RuntimeObject01" );
	if( pCtor ) {
		IObject* pObj = pCtor->Construct();
		pObj->GetInterface( &m_pUpdateable );
		m_ObjectId = pObj->GetObjectId();
	}
	return true;
}
```
- RuntimeObjectSystem::Initialise