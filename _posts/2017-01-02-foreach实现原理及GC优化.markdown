---
layout:     post
title:      "Foreach实现原理及性能分析"
subtitle:   "foreach、GC及协程"
date:       2017-01-02
author:     "Luffy"
header-img: "img/BG/background1.jpg"
catalog: true
tags:
    - U3D
    - c#
    - 性能优化

---


## 前言

考虑以下在U3D中的代码：

~~~cs
public List<int> testList = new List<int>();
	void Start()
	{
		for(int i = 0; i<10; i++)
			testList.Add(i);
	}
	// Update is called once per frame
	void Update () 
	{
		int testInt;
		//使用for循环
		for(int i = 0; i < testList.Count;i++)
		{
			testInt = i;
		}
		//使用foreach循环
		foreach(int i in testList)
		{
			testInt = i;
		}
	}
~~~

通过分别使用for和foreach进行循环操作，会发现，在使用foreach进行循环迭代时，会在每帧产生40B的GC Alloc。产生这种现象的原因是什么？如何进行优化？下文将详细讲解。

在[这里](http://stackoverflow.com/questions/18718399/every-iteration-of-every-foreach-loop-generated-24-bytes-of-garbage-memory)有相关问题的相关讨论。

## Foreach实现原理
*Foreach*是c#的语法糖，通过封装迭代过程的实现细节，可以更加便捷的实现迭代。以下，参考自[coderbetter](http://codebetter.com/davidhayden/2005/03/08/implementing-ienumerable-and-ienumerator-on-your-custom-objects/)一篇文章来探索foreach实现细节。

~~~cs
foreach(Student student in myClass)
Debug.Log(student);
~~~

为实现foreach，myClass需要实现__IEnumerable__接口，其中需要实现`public IEnumerator GetEnumerator()`，而*IEnumerator*接口需要实现:    

  * public object Current;  
  * public void Reset();
  * public bool MoveNext();

而其中IEnumerator的实现如下所示：

~~~cs
	private class ClassEnumerator : IEnumerator
		{
	    private ClassList _classList;
        private int _index;
        public ClassEnumerator(ClassList classList)
        {
            _classList = classList;
            _index = –1;
        }
        public void Reset()
        {
            _index = –1;
        }
        public object Current
        {
            get
            {
                return _classList._students[_index];
            }
        }
        public bool MoveNext()
        {
            _index++;
            if (_index >= _classList._students.Count)
                return false;
            else
                return true;
        }
	}
~~~
*Foreach*语法糖可以更方便的进行迭代，否则需要手动实现IEnumerable和IEnumerator接口。

## Foreach与GC
回到开头的问题，为什么在使用foreach时会产生额外的GC呢？参考自[这篇博客](https://codingadventures.me/2016/02/15/unity-mono-runtime-the-truth-about-disposable-value-types/)，可以得出为什么在U3D中会出现这种问题。    
先假设有这样一段代码：

~~~cs
void Update()
{
	var fibonacci = new List<int>{1,1,2,3,5,8,13};
	foreach(var f in fibonacci)
		Debug.Log(f);
}
~~~

__Foreach__在实现时，等同于：

~~~cs
using(var enumerator = fibonacci.GetEnumerator())
{
	while(enumerator.MoveNext())
   {
     var current = enumerator.Current;
     Console.WriteLine(current);
   }
}
~~~

而在老版本的mono编译器中，将*__using__*中的表达式，例如`using (ResourceType resource = expression) `转换为：

~~~cs
{
	ResourceType resource = expression;
	try
	{
	   statement;
	}finally{
		((IDisposable)resource).Dispose();
	}
}
~~~
而将resource（值类型）转换为*__IDisposable__*接口类型时，会发生装箱操作，这也就是引起GC的原因。

#### c#编译器优化
在新版本的编译器中，会判断resource类型，若为值类型，则直接`finally{resource.Dispose();}`避免装箱操作。关于具体的*__using相关转换__*,可参考[这篇博文](https://ericlippert.com/2011/03/14/to-box-or-not-to-box/)。

##StartCoroutine
U3D中`StartCoroutine`也利用了迭代器的原理，yield关键字使程序在确定时间点停止给定时间-->保存状态-->一定时间后继续运行。使用协程可以实现延迟操作。在如_pathfinding_等操作，分为多帧运行可以缓解运行压力。
`yield`关键字实现通过[使用状态机实现](https://blogs.msdn.microsoft.com/oldnewthing/20080812-00/?p=21273)。

如:

~~~cs
public IEnumerable<int> CountFrom(int start)
{
	for(int i = start;i<=limit;i++)
		yield return i;
}
~~~

内部转换：

~~~cs
public bool MoveNext()
{
	switch(state $0)
	{
		case0:goto resume$0;
		case1:goto resume$1;
		case2:goto false;
	}
	resume$0:;
	for(int i=start;i<=this.$0.limit;i++)
	{
		curent$0 = i;
		state&0=1;
		return true;
	}
	resume$1:;
}
~~~
大概思路如上所示，通过状态机来记录程序运行的位置，已完成yield关键字操作。

在U3D中，`StartCoroutine`吸收IENumerator和yield核心思想，实现延迟操作。在stackoverflow上，有[一篇关于实现原理的分析](http://stackoverflow.com/questions/12932306/how-does-startcoroutine-yield-return-pattern-really-work-in-unity)。


## 总结
从`Foreach`出发，分析实现原理引出IENumerator实现和yield的原理。最后，在给出关于`StartCoroutine`实现原理的一篇文章。




