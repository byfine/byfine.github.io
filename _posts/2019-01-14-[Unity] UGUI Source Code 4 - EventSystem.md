---
layout: post
title: UGUI源码分析 4 - 事件系统
description: "UGUI源码，事件系统"
modified: 2019-01-14
tags: [Unity]
---

![]({{ site.url }}/images/post/UGUI-Source/EventSystem1.jpg)

## 点击按钮过程
我们通过点击一个Button组件为例，了解一下UGUI怎么实现点击检测的。

首先我们知道要实现点击等交互功能，场景中必须有 EventSystem 组件。可以看到其上挂载了EventSystem和 StandaloneInputModule 两个脚本。

在 EventSystem 的 Update() 函数中，m_CurrentInputModule.Process() 调用了InputModule的执行函数，而在 StandaloneInputModule 中则通过override对其进行了具体的实现：  
![]({{ site.url }}/images/post/UGUI-Source/EventSystem2.jpg)  

我们查看 ProcessMouseEvent()，最终会调用ProcessMousePress函数，而这个函数中就进入了点击检测的关键：  
![]({{ site.url }}/images/post/UGUI-Source/EventSystem3.jpg)  
该函数会执行ExecuteEvents脚本的 Execute静态函数。

Execute函数：  
![]({{ site.url }}/images/post/UGUI-Source/EventSystem4.jpg)  

我们可以知道，pointerEvent.pointerDrag 就是我们点击的物体，pointerEvent是 eventData，而 ExecuteEvents.pointerClickHandler 则是点击委托，我们在ExecuteEvents中可以看到其定义：  
![]({{ site.url }}/images/post/UGUI-Source/EventSystem5.jpg)   

回头查看Execute函数，从而我们了解到，GetEventList 会在target 物体上查找继承了 IEventSystemHandler 接口的组件，然后对每个组件执行委托函数。对于此时的点击事件，就执行上面的 pointerClickHandler。

综上，我们可以知道，只要组件继承了 IEventSystemHandler， 在不同情况（点击、抬起、拖动等）触发时，就会执行对应接口的相应委托事件，从而实现事件交互。

## 获取组件
我们再思考一下，为什么EventSystem可以知道我们点击到了组件，或者说事件系统是如何获得点击的物体的。

我们回头看 StandaloneInputModule 中的 ProcessMouseEvent 函数，其中第一句执行了 var mouseData = GetMousePointerEventData(id); ，进入函数可以知道这是其父类实现的一个函数。  

在其中我们看到一个熟悉的函数，eventSystem.RaycastAll(leftData, m_RaycastResultCache);  之前我们就说过，事件点击的本质就是射线检测，因此很可能这就是实现射线检测的地方。 进入后可以看到其中调用了 modules 的 Raycast 函数， 而modules就是 挂载在 Canvas 上的 GraphicRaycaster 组件，这样解释了为什么没有此组件也无法实现点击交互。

最终我们到达了射线检测的核心部分，Raycast函数。先根据Canvas Render Mode 及Block相关设置进行射线检测。然后调用了重载函数Raycast，遍历Canvas下每个Graphic组件，把鼠标位置点击到的组件加入列表。进行 ignoreReversedGraphics 的操作。最后获取每个组件的信息。

ps. 通过 GraphicRaycaster 源码可以详细了解 Blocking Objects 和 Blocking Mask的作用。通过 blockingObjects 和 m_BlockingMask 的设置进行射线检测，会获得一个hitDistance，这个距离是到阻塞物体的距离。之后又会计算物体到摄像机的距离 distance， 如果distance 大于 hitDistance，这个物体就不算检测到，从而实现阻塞。

整个过程如下：  
![]({{ site.url }}/images/post/UGUI-Source/EventSystem6.jpg)   