---
layout: post
title: Shader：Forward Rendering Path
description: "Forward Rendering Path 简介"
modified: 2015-11-26
tags: [Shader, Unity3D]
---

Forward Rendering Path 根据影响物体的灯光，在一个或多个通道中渲染每个物体。根据每个灯光的设置和强度，每个灯光也被Forward Rendering不同对待。

###实现细节
在正向渲染中，一些影响物体的最亮的灯会被逐像素模式渲染。然后，最多4个点光会逐顶点渲染。其他灯光则被计算为Spherical Harmonics(SH)，这样速度更快但只是一种近似值。一个灯光是否是逐像素灯光取决于：

- 渲染模式设置为Not Important的灯光始终是逐顶点或SH。
- 最亮的灯光总是逐像素的。
- 渲染模式为Important的灯光总是逐像素的。
- 如果上面的总结出的灯光数量少于Quality Setting中设置的Pixel Light Count，那么多出数量的灯光会按亮度减少的顺序逐像素渲染。

对每个物体的渲染如下：

- 根据Pass应用一个逐像素方向光和所有的逐顶点/SH光。
- 其他逐像素光是在额外的pass渲染，每个pass对应一个灯光。

比如，有一些物体收到多个灯光（A-H）影响的物体：

![]({{ site.url }}/images/post/Shader/Forward-Rendering-Path-1.png)

我们假设灯光A-H都是同样的颜色和强度并且都设置为Auto rendering mode，所以它们都只会按照距离物体的远近来排序。最亮的光会被渲染为逐像素模式（A-D），然后最多4个光为逐顶点（D-G），然后最后剩下的光是SH（G-H）：

![]({{ site.url }}/images/post/Shader/Forward-Rendering-Path-2.png)

注意灯光组的重叠，比如最后一个逐像素光源也以逐顶点光照模式的方式渲染，这样能减少当物体和灯光移动时可能出现的"光照跳跃"现象。

###Base Pass
Base pass使用一个逐像素方向光和所有SH光渲染物体。此通道还负责渲染着色器中的 lightmaps, ambient 和 emissive lighting。在此通道中渲染的方向光可以产生阴影。需要注意的是，使用了光照贴图的物体不会得到SH光的光照。

###Additional Passes 
附加通道用于渲染影响物体的其他逐像素光源。这些通道中渲染的光源无法产生阴影（因此， Forward Rendering只支持一个能产生阴影的方向光）。

###性能考量
渲染球面调和（SH）光很快。它们只花费很少的CPU计算时间，并且实际上无需花费任何GPU计算时间（换言之，基础通道会计算所有SH光照，但由于SH光的计算方式，无论有多少SH光源，计算它们所花费的时间都是相同的）。

球面调和光源的缺点有：

- 它们计算的是物体的顶点而不是像素。这意味着它们不支持light Cookies和normal maps。
- 球面调和光只有很低的频率。球面调和光不能产生锋利的照明过渡。它们也只会影响散射光照（对高光来说，球面调和光的频率太低了）。
- 球面调和不是局部的，靠近曲面的球面调和点光和聚光可能会"看起来不正确"。

总的来说，球面调和光的效果对小的动态物体来说已经足够好了。

---			
> 参考：<br>
[Forward Rendering Path Details](http://docs.unity3d.com/Manual/RenderTech-ForwardRendering.html)<br>