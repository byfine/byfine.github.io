---
layout: post
title: OpenGL 简介
description: "OpenGL 简介"
modified: 2015-10-12
tags: [OpenGL]
---

下面一系列OpenGL笔记，都是[LearnOpenGL-CN](http://learnopengl-cn.readthedocs.org/zh/latest/)的学习笔记。只是我个人的要点总结，并不详细，具体内容请参考原网站内容。

###概念
OpenGL被认为是一个**应用程序编程接口**(Application Programming Interface, API)，它包含了一系列可以操作图形、图像的方法。然而，OpenGL本身并不是一个API，仅仅是一个规范，由[Khronos组织](http://www.khronos.org/)制定并维护。
实际的OpenGL库的开发者通常是显卡的生产商。


###核心模式(Core-profile)与立即渲染模式(Immediate mode)
早期的OpenGL使用**立即渲染模式**(也就是固定渲染管线)，这个模式下绘制图形很方便。但OpenGL的大多数功能都被库隐藏起来，开发者很少能控制OpenGL如何进行计算。

从OpenGL3.2开始，规范书开始废弃立即渲染模式，推出**核心模式**，这个模式完全移除了旧的特性。当使用核心模式时，OpenGL迫使我们使用现代的做法。现代做法要求使用者真正理解OpenGL和图形编程，它有一些难度，然而提供了更多的灵活性，更高的效率，更重要的是可以更深入的理解图形编程。

所有OpenGL的更高的版本都是在3.3的基础上，添加了额外的功能，并不更改进核心架构。新版本只是引入了一些更有效率或更有用的方式去完成同样的功能。因此所有的概念和技术在现代OpenGL版本里都保持一致。

###扩展(Extension)

OpenGL的一大特性就是对扩展的支持，当一个显卡公司提出一个新特性或者渲染上的大优化，通常会以扩展的方式在驱动中实现。通过这种方式，开发者不必等待一个新的OpenGL规范面世，就可以方便的检查显卡是否支持此扩展。
{% highlight C++ %}
if(GL_ARB_extension_name)
{
    // 使用一些新的特性
}
else
{
    // 不支持此扩展: 用旧的方式去做
}
{% endhighlight %}

###状态机(State Machine)

OpenGL自身是一个巨大的状态机：一个描述OpenGL该如何操作的所有变量的大集合。
OpenGL的状态通常被称为OpenGL**上下文**Context)。我们通常使用如下途径去更改OpenGL状态：设置一些选项，操作一些缓冲。最后，我们使用当前OpenGL上下文来渲染。

用OpenGL工作时，我们会遇到一些状态设置函数(State-changing Function)，以及一些在这些状态的基础上状态应用的函数(State-using Function)。只要你记住OpenGL本质上是个大状态机，就能更容易理解它的大部分特性。


###对象(Object)
在OpenGL中一个对象是指一些选项的集合，代表OpenGL状态的一个子集。可以把对象看做一个C风格的结构体。
使用对象的一个好处是我们在程序中不止可以定义一个对象并且设置他们的状态，在我们需要进行一个操作的时候，只需要绑定预设了需要设置的对象即可。

###原始类型(Primitive Type)

使用OpenGL时，建议使用OpenGL定义的原始类型。OpenGL定义的这些GL原始类型是平台无关的内存排列方式。比如使用float时加上前缀GL(GLfloat)。int，uint，char，bool等等类似。

---
> 参考：
[OpenGL](http://learnopengl-cn.readthedocs.org/zh/latest/01%20Getting%20started/01%20OpenGL/)