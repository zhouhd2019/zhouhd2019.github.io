---
layout:     post
title:      "Runtime Compiled C++ for Rapid AI Development"
subtitle:   "GameAIPro Chapter15"
date:       2020-09-15  17:15:00
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
##### 初始化
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
- RuntimeObjectSystem::Initialise，主要调用RuntimeObjectSystem::SetupObjectConstructors，工作有两部分，RuntimeObjectSystem::SetupRuntimeFileTracking和ObjectFactorySystem::AddConstructors
- RuntimeObjectSystem::SetupRuntimeFileTracking，设置运行时文件监控。循环处理每个Constructor，将文件记录到RuntimeObjectSystem，清除老的头文件依赖记录，遍历当前Constructor的trackingInfo，记录头文件和动态链接库，如果是可以编译的文件，则会同时加入监控，文件被修改时会调用回调函数，实现动态编译CPP
- ObjectFactorySystem::AddConstructors，将Constructors注册到ObjectFactorySystem。需要进行新老Constructors替换，替换流程比较复杂：(1)序列化老对象，每种Constructor记录了自己当前有效对象，分别对它们调用序列化；(2)构造新对象，记录新Constructor，如果有老Constructor，则调用新Constructor创建相应数量的新对象；(3)反序列化，将之前的数据存到新对象；(4)构建并初始化单例对象，对象定义时就已经指定是不是单例以及是否需要自动初始化；(5)新对象初始化，调用init，以及可选的测试一下序列化函数；(6)删除老对象
- ConsoleGame::Init第二部分主要展示了如何创建动态对象和进行相关设置，先通过RuntimeObjectSystem->GetObjectFactorySystem->GetConstructor来获取RuntimeObject01的Constructor，调用Construct来构建一个对象，通过GetInterface获取可以动态编译的函数，保存函数指针和对象ID
- RuntimeObject01继承于TInterface<IID_IUPDATEABLE, IUpdateable>，作为示例只定义了Update函数，类定义结束后有一个宏定义REGISTERCLASS(RuntimeObject01)。宏定义实现了自动定义TObjectConstructorConcrete等等
- IID_IUPDATEABLE只是一个数字，用来区分可以动态编译的不同函数，这个数字通过模板传递到GetInterface，这样GetInterface就可以通过不同的IID来返回不同的函数，如果传入的是类当前的IID，就返回类的函数，不是则让父类处理，RuntimeObject01对应父类是IUpdateable。这种设计可能有些简单，每个类只能有一个对外接口
- TObjectConstructorConcrete继承于IObjectConstructor，后者定义了大量接口。TObjectConstructorConcrete在REGISTERCLASS时会将自身注册到PerModuleInterface，来让RuntimeObjectSystem等系统获取Constructor。TObjectConstructorConcrete::Construct函数构造一个新的对象，并将这个对象记录下来

##### 运行时修改
- 文件改变时如何重新编译，调用WatchCallback，将被修改了的文件路径记录下来，尝试触发重新编译。调用RuntimeObjectSystem::OnFileChange，记录修改过的文件的依赖，最后开始重新编译。调用RuntimeObjectSystem::StartRecompile，记录需要的所有文件，如需要链接的库和重新编译的源文件，调用BuildTool::BuildModule。在Windows上时查找VS路径，组装命令，最后调用编译命令
- 如何重新载入编译后的代码，ConsoleGame::MainLoop会检查是否触发了编译并且编译完毕，是的就调用RuntimeObjectSystem::LoadCompiledModule，通过LoadLibraryA载入编译好的文件，GetProcAddress来获得GetPerModuleInterface函数指针，这样就可以获得这个模块的相关信息
- 如何使用编译后的代码，每个类固定的接口都是通过函数指针来调用的，重新编译后，会触发逻辑代码的OnConstructorsAdd，逻辑代码需要实现这个函数，重新获取接口函数的指针。另外调用时也不能直接调用具体函数，必须通过函数指针来调用