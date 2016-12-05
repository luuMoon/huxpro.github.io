---
layout:     post
title:      "U3D性能优化学习笔记"
subtitle:   "性能测试工具学习"
date:       2016-12-04
author:     "Luffy"
header-img: "img/BG/topbackground.jpg"
catalog: true
tags:
    - Unity3D
    - 性能优化

---


## 前言
在游戏开发过程中遇到刷新界面卡顿等现象，决定学习一下Unity相关的优化工作。首先要做的就是了解Unity相应的性能测试工具以找到性能的瓶颈。依据[U3D官方文档](https://unity3d.com/cn/learn/tutorials/topics/performance-optimization/profiler-window?playlist=44069)和google上的一些文档，熟悉相关的性能测试工具。

## Profiler window

 
![profiler截图](img/U3D/Performance/profiler1.png)    
  
U3D内置的性能分析器，可以分析游戏中各个模块相应的性能，找出性能瓶颈并修复相应问题.

![profiler控制条](img/U3D/Performance/profilerBar.png)

profiler的工具条可以控制Profiler的相关显示操作。可以通过`Add Profiler`来添加相应的监测窗口。`Record`为是否显示性能分析数据。`Deep Profile`开启时，所有的函数调用都将会被检测，这在调试关键性能代码时十分有用。⚠**_开启时会造成巨大开销，可能造成内存耗尽。_**

同时，你也可以使用`Profile.BeginSample`和`Profile.EndSample`来手动检测代码性能。

```
Profiler.BeginSample ("testProfilerSample");
for (int i = 0; i < 1000; i++)
{
	testString.Append (i);
	if(i == 999)
	{
		Debug.Log(testString);
	}
}
Profiler.EndSample ();
```
之后就可以在Profiler Window中通过testProfilerSample标签来查看相应代码的性能分析，需要注意的是：只有在相应帧上才可以观察到。


Profiler有几个不同的分析监测部分:    
>   * CPU 
>    1. Scripts           
>    2. Rendering
>    3. Phsics
>    4. GC
>    5. VSync
>    6. GI
>    7. Others 
> * Rendering
>    1. Scripts     
>	  2. Texture Memory    
>	  3. GC    
>	  4. Material Count
> * Memory    
> * Others

通过Profiler窗口可以观察到每一帧当中，相应操作所消耗的时间。首先简单了解一下[帧率(Frame Rate)](https://en.wikipedia.org/wiki/Frame_rate),帧率即是图像渲染到屏幕上的速度，单位是FPS。而当我们在检测游戏性能时，需要关注每一帧所消耗的时间(ms),才能够更加准确的测量并改进相关的游戏性能，同时也可以更加精准的找到每一帧当中不同函数所消耗的具体时间。

例如，通过CPU监视窗口可以观察到一帧当中脚本、GC和VSync等所消耗的时间，找到性能瓶颈。
![Profiler CPU](img/U3D/Performance/profilerCPU.png)

不同部位的性能分析通过不同的颜色显示，可以点击以开关，拖拽以排列显示顺序。同时，在Hierarchy窗口中，可以通过点击工作条中不同的信息（如Time ms），来切换信息排列方式。

其中，WaitForTargetFPS是当帧率快于屏幕刷新率或设定的帧率值时等待的耗费时间。参考[这篇文章](http://weibo.com/p/1001603954695990318082#_loginLayer_1472091401699)可以帮助更好的理解WaitForTargetFPSGfx.WaitForPresent和 Graphics.PresentAndSync。

Timeline View是另一种观测试图。可以更好的了解CPU任务的排列顺序和负责相应任务的线程。
![TimeLine](img/U3D/Performance/profilerTimeline.png)

在Rendering窗口中，显示的是Unity实时渲染状态，同时在Unity的Game窗口右上角也有一个类似的stats状态信息可以查看。
![profiler状态信息stats](img/U3D/Performance/profilerStats.png)

信息包含有:   
     
* FPS
* Batches  
* Tris/Verts
* 等等

其中，Batches和setPass calls为批处理数量和pass数量，数量过大会引起CPU占用率过高，可以查看[Unity官方手册](https://docs.unity3d.com/Manual/DrawCallBatching.html) 做进一步了解。

## 总结
通过学习了解U3D的Profiler相关信息，了解如何使用自带的性能分析器分析游戏的相关瓶颈点，之后可以采取不同的性能优化策略，如减少GC、优化DrawCall和GPU的相关优化。
 










