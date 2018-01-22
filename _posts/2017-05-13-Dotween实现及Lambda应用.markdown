---
layout:     post
title:      "Dotween及Lambda应用"
subtitle:   "Dotween的实现及其中Lambda的作用"
date:       2017-05-13
author:     "Luffy"
header-img: "./img/BG/topbackground.jpg"
catalog: true
tags:
    - C#
    - U3D
    - 工具类

---

## 前言

在使用Dotween时，其中用到了Lambda表达式。好奇匿名表达式原理和特点，如：闭包（最近看Lua--小本本先记下)。 同时在C#中有大量的使用匿名函数的场景，于是花了点时间研究相关知识点。

----
2018/01/22

好久前写的，之前没有太多时间去写，结果就耽误了。现在看来这几点又有些没什么需要写的，那就简单的加上点网站索引吧。

#### Dotween

至于为何项目中要使用Dotween，之前也介绍过，根据比较，Dotween与同类的插值软件相比，效率更高，使用起来也较为方便，支持c#扩展，可视化(__好像要付费__)等。

附上[Dotween官方教程网址](http://dotween.demigiant.com/documentation.php)。

#### Lambda及闭包

lambda和闭包。 lambda[根据官方解释](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/statements-expressions-operators/lambda-expressions),可以被当作匿名函数使用，而匿名函数则是为了方便程序可以将函数作为变量传递，更加方便灵活。

而闭包，[参考自c# in depth](http://csharpindepth.com/Articles/Chapter5/Closures.aspx) : `closures allow you to encapsulate some behaviour, pass it around like any other object, and still have access to the context in which they were first declared.` 。 与方法并没有本质区别，但储存了声明时的环境上下文。不注意会出现一个BUG(网上很多说明)。

#### Lambda与Linq

Linq大量运用Lambda，但项目中不推荐使用，关于Linq对性能的影响，论[坛中也有许多人对此发表了意见](https://forum.unity.com/threads/to-linq-or-not-to-linq.223887/)。

同时，也有人给出了MSDN和Unity对Linq的性能分析，最后的看法便是，项目中尽量避免使用Linq。

MSDN文章 ： [文章地址](https://developer.microsoft.com/en-us/windows/mixed-reality/performance_recommendations_for_unity)

Unity分析GC文章 ： [文章地址](https://unity3d.com/de/learn/tutorials/topics/performance-optimization/optimizing-garbage-collection-unity-games)。

