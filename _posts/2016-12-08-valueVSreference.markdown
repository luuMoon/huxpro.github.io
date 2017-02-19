---
layout:     post
title:      "值与引用类型的转换"
subtitle:   "装箱拆箱、string、统一类型"
date:       2016-12-08
author:     "Luffy"
header-img: "./img/BG/topbackground.jpg"
catalog: true
tags:
    - 基础知识
    - c#

---


## 前言
在学习U3D的GC性能分析时，影响GC性能的一个因素是装箱操作。频繁的装箱如**_foreach_**操作会使GC更加频繁导致游戏性能变差。而装箱的核心概念是c#值类型与引用类型的转换，所以今天学习一下相关基础知识。

## 值类型与引用类型
#### 统一类型系统
c#提供一个统一类型系统，在c#语言中，所有的数据类型都继承自同一个跟类型**_object_**。统一类型的好处有很多，例如可以保证类型安全、可以定义通用的类型函数。下面介绍一下object相关定义。

 
* Equals(Object) : 判断与给定物体是否相等
* Finalize()     : 在GC前释放资源等    
* GetHashCode()  : 最快时间对物体进行简单判断    
* ToString()     : 返回代表物体的string值    
* GetType()      : 返回物体类型（Type是反射的基础，与metadata相关，学习反射时再做详解）

在c#中，所有类型都继承自*object*,即c#为统一类型系统，所有的类型，可以使得无论是int等基本类型，还是自定义的class类型都有一些共用的行为。
#### 值与引用
值和引用是c#中的两种类型，都继承自Object，所以可以互相转换。其中，将值类型转换为引用类型的操作称为*__boxing__*,反之为*__unboxing__*.
首先来了解并区分一下值类型和引用类型，参考自[*梦里花落知多少*](http://www.cnblogs.com/anding/p/5229756.html)的博客。  
同时根据[*MSDN*](https://msdn.microsoft.com/en-us/library/t63sy5hs.aspx)，值类型即是内存中存储自身的数据内容，而引用类型则是一个指向另一个保存数据内容的地址。其中，值类型存储在Stack中，当值不在其作用域时自动释放，而引用类型存储在托管堆中，分配和释放由GC来管理。具体数据类型分布如下：

|值类型 | 引用类型 | 
|:-------:| :------:|  
| 所有数值类型| *String*|
|布尔值、Char|数组|
|自定义类型：struct、enum|Class|
|Data等 | Delegate|


`String`类型作为引用类型，存储在托管堆中，不过其具有一些非常规的使用方式，将在以后介绍。 
  
同时，关于`String`和`string`大小写的区别，可以参考[stackOverFlow的这篇文章](http://stackoverflow.com/questions/7074/what-is-the-difference-between-string-and-string-in-c/215422#215422)。

也有一些程序元素不能定义为类型，如下：  

* Namespace       
* Event
* 属性
* constants等

#### ref和out
函数参数赋值默认为值传递，如下为在U3D中测试代码,输出结果为10，10。无论是值类型(testValue)还是引用类型(ValueClass`传递Stack中引用类型的地址值`),都是按值传递。因为`String`是一种特殊的引用类型，下文将详细解释其特殊性。

~~~cs
public class refVSout : MonoBehaviour 
{
    public class ValueClass
    {
        public int mValue;
    }

    public string referenceType = "Before";
    public void valueTransfer(int param1 , ValueClass param2)
    {
        param1 += 10;
        param2.mValue += 10;
    }

    void Start () 
    {
        int testValue = 10;
        ValueClass valueClass = new ValueClass ();
        valueTransfer (testValue, valueClass);
        Debug.Log (testValue + "  " + valueClass.mValue);
    }

}
~~~
添加__*ref和out*__关键字可以按引用传递参数。其不同是，ref参数物体需要在传递前初始化，而out为传递后进行初始化（且不允许在离开函数时未初始化）。具体区别可以参考[StackOverFlow](http://stackoverflow.com/questions/388464/whats-the-difference-between-the-ref-and-out-keywords)。

## 装箱与拆箱
关于为什么需要装箱与拆箱，在[StackOverFlow上有一个不错的解释](http://stackoverflow.com/questions/2111857/why-do-we-need-boxing-and-unboxing-in-c)。
既然在C#中存在着valueType和referenceType，那么在两种类型转换的同时就会有装箱与拆箱的操作。根据[MSDN相关介绍](https://msdn.microsoft.com/en-us/library/yz2be5wk.aspx),装箱即是将值类型转化为*__object__*或*__接口__*的过程。

~~~cs
int i = 123;
object o = i; //boxing
~~~

装箱在GC堆中创建新的*__Object__*,然后将值类型的值复制到这个Object中。如下图所示：
![boxingTexture](/img/CS/boxing.gif)

拆箱与装箱相反，是引用类型转换为值类型的过程。

~~~cs
int i = 123;
object o = i; //boxing
int j = (int)o; //unboxing
~~~
拆箱操作首先判断__Object__是否可以转换为指定的值类型，然后将值复制到新的值类型变量当中。
![boxingTexture](/img/CS/unboxing.gif)

在理解了装箱和拆箱的相关概念之后，就可以理解什么是箱子了，同样参考自[*梦里花落知多少关于装箱拆箱的博客*](http://www.cnblogs.com/anding/p/5236739.html)，箱子即是一个存放了值类型字段的引用对象实例，并存储在托管堆中。所以*__只有值类型才有拆箱和装箱这两种状态，引用类型一直都在箱子中__*。

装箱的操作由于需要在托管堆中申请内存，所以十分消耗性能，频繁的装箱也会照成托管堆中内存被分为许多小块，所以需要避免频繁装箱，尤其是隐式装箱。如`int x = 100; ArrayList a = new ArrayList(3); a.Add(x);`，由于Add的方法定义为`public virtual int Add(object value)`，在将x转换为object时发生装箱问题。

而关于如何优化游戏中的GC等问题，以后会专门花时间写一篇博客。


## 总结
介绍了在c#中值类型和引用类型的相关概念，和两者之间转换时发生的装箱与拆箱操作，通过减少装箱操作可以达到优化相关游戏性能的目的。

