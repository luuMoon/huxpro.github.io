---
layout:     post
title:      "LuaFramework工作流程(一)"
subtitle:   "Lua的工作原理及加载流程"
date:       2017-06-17
author:     "Luffy"
header-img: "./img/BG/topbackground.jpg"
catalog: true
tags:
- Lua
- U3D
- 语言
- 热更新

---

## 前言
在新的Momo项目中使用Lua实现脚本热更，最终决定采用ToLua  ----> [Github地址](https://github.com/topameng/tolua)。  用两篇BLOG分别谈一下Tolua相关例子及Lua相关使用方法；然后再介绍LuaFramework的使用流程。

## Lua及热更新简介

关于热更新，[这篇BLOG](http://www.cnblogs.com/crazylights/p/3884650.html)简单介绍了热更的含义及游戏中使用热更的目的,简单的说就是减短玩家获取最新资源的流程与时间，从而改善游戏体验。而对于正常的资源热更，以往使用的是Unity自带的AssetBundle，通过将所需要更新的资源根据类型(如Shader打入一个包)等打包放入指定服务器(*第一次放入StreamingAssetsPath*),在游戏启动时，自动更新。

而代码作为一种特殊的资源无法使用AB来进行热更，代码热更的方案和所使用的语言也并不固定，项目中采用了最为通用的Lua来实现这一目的。

`Lua`作为一种嵌入式脚本语言，由于可以利用*Lua虚拟机* ， 将方法列入一等公民(First-class), 使得语言更加适合于游戏热更。

关于Lua的学习，可以选择[Programming in Lua](https://www.lua.org/pil/)。

这里大概介绍一下Lua的基础知识，根据Programming In Lua， 大概可以分为三部分来学习：基础的语法，难点在于闭包与协程（闭包涉及到GC问题，UWA也有一篇相关BLOG，待写) ；  `Table`,Lua中使用Table来表示所有的数据结构，可以简单理解其为`关联数组`; 最后一部分是lua与c的互调。

学习过程中一些资料    
	1. Programming in lua 课后习题答案： [地址](https://github.com/bobeff/playground/tree/master/programming-in-lua)    
	2. [SOF关于闭包的解析](http://stackoverflow.com/questions/36636/what-is-a-closure)    
	3. 使用VS2013来配置下载的LuaC文件： [BLOG地址](http://www.cnblogs.com/jevil/p/4832938.html)    	
	这里的lua与C互调，是通过c中虚拟的一个栈结构，而c#与lua互调，是通过调用DLL使用Tolua库(c语言),再调用lua。  	    
	
## Tolua例子简介

在下载Tolua之后，在目录`Examples`中包含20多个例子。lua可以通过读取string和文件来将lua加载进lua虚拟机。

```cs
//新建lua虚拟机
LuaState lua = new LuaState();
lua.Start();
string hello =
    @"                
    print('hello tolua#')                                  
    ";
lua.DoString(hello, "HelloWorld.cs");
```

通过新建lua虚拟机，并使用DoString来将lua加载进虚拟机。

也可以通过`lua.AddSearchPath(fullPath);` 加载进lua搜索路径，然后通过DoString加载进虚拟机。  在这里，lua的require加载模型，[相关机制](https://www.lua.org/pil/8.1.html)。

接下来的几个例子，基本是在介绍如何使用ToLua来完成c#与Lua的互调，如调用lua函数：

~~~cs
 func = lua.GetFunction("test.luaFunc");
 func.BeginPCall(); 
 //param               
 func.Push(123456);
 func.PCall();        
 int num = (int)func.CheckNumber();                    
 func.EndPCall();
 return num;      
~~~

如调用Lua协程 : `通过LuaLooper脚本模拟Update/FixedUpdate`调用Lua，实现Lua协程。

其他的一些例子如调用json、ProtoBuffer、GameObject等也是多有类似，具体可在项目中测试学习。

## 小结
介绍了Lua及Tolua， 还有一些如：`Lua实现OO` ， `Protobuf应用` , `LuaFramework应用` 等，留在下节介绍。

	
	
	