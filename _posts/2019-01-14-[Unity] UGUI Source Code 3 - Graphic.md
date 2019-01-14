---
layout: post
title: UGUI源码分析 3 - 图形绘制过程
description: "UGUI源码，图形绘制过程"
modified: 2019-01-14
tags: [Unity]
---

## 绘制过程
![]({{ site.url }}/images/post/UGUI-Source/Graphic.png)

以Image为例进行分析。每次对图片进行更改的时候，都会调用 SetVerticesDirty 函数，在其中最主要是调用此函数 CanvasUpdateRegistry.RegisterCanvasElementForGraphicRebuild(this);   

这个函数会将绘制的元素加入到一个rebuild队列 m_GraphicRebuildQueue，对其的操作主要在 PerformUpdate 函数中进行，PerformUpdate 是一个注册到 Canvas.willRenderCanvases 的事件，根据官方手册描述，该事件会在Canvas rendering 前发生。  
![]({{ site.url }}/images/post/UGUI-Source/Graphic5.png)  


这是PerformUpdate中对 m_GraphicRebuildQueue队列的操作：  
![]({{ site.url }}/images/post/UGUI-Source/Graphic2.jpg)   

其中元素的 Rebuild函数是 ICanvasElement接口定义的，具体的实现位置是在 Graphic类中:  
![]({{ site.url }}/images/post/UGUI-Source/Graphic3.jpg)  

而在 UpdateGeometry() 中的函数 则执行了绘制的核心功能，类似我们上个例子中实现的那样，进行创建网格，设置参数等操作。
其中， OnPopulateMesh 是填充网格的函数，并且是虚函数，Image继承自Graphic，对其进行了override，可以看到，Image 分别对不同 Filltype 进行了不同的网格绘制：  
![]({{ site.url }}/images/post/UGUI-Source/Graphic4.jpg)   

Canvas Update 的过程：  
![]({{ site.url }}/images/post/UGUI-Source/Graphic6.jpg)  

另外，在 UpdateMaterial() 函数中，则对进行的材质、贴图进行了设置。

