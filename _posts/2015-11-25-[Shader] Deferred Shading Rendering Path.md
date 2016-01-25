---
layout: post
title: Shader：Deferred Shading Rendering Path
description: "Deferred Shading Rendering Path 简介"
modified: 2015-11-25
tags: [Shader, Unity3D]
---

###概览
当使用Deferred Shading，对于灯光可以影响的物体数量没有限制。所有灯光基于每个像素评估，即意味着它们都与法线贴图交互正常等。所有灯光都可以有cookies 和阴影。

Deferred shading的优势是灯光处理开销是和灯光照射到的像素数成比例的。这是由场景中灯光体积决定的，而与灯光照射多少物体无关。因此，让灯光小一些可以提高性能。Deferred shading也拥有高度统一和可预测的行为。每个灯光是逐像素计算的，所以在大三角形面上也不会有灯光计算失误。

但是另一方面，deferred shading并不真正支持抗锯齿，也不能处理半透明物体（这是在forward rendering中处理）。也不支持Mesh Renderer的Receive Shadows标志，而且对于culling masks也有限制。只能使用4个culling masks。意味着 ，你的遮挡层最多只有四个，即28-32的layers 必须被设置。

###要求
需要图形卡支持Multiple Render Targets（MRT），Shader Model 3.0及以后，支持深度渲染贴图和双面模板缓存。大部分2005年之后的显卡都支持延迟光照。对于移动平台，支持更有限因为MRT格式的使用。

注意：延迟渲染不支持Orthographic 投影。

###性能考量
前面说过，deferred shading中灯光的渲染开销与灯光照射的像素数有关而与场景的复杂度无关。所以小的点光或聚光开销很少，而如果它们被场景物体全部或部分挡住，开销会更少。

当然，使用阴影的灯光更加消耗性能。在deferred shading中，对每个阴影投射灯光，阴影投射的物体仍需要被渲染一次或多次。此外，应用了阴影的灯光shader拥有更多渲染开销。

###实现细节
Deferred shading的基本思想是分两步进行：第一步，场景在不进行光照模型运算的情况下渲染，然后将每个像素的位置、法线、高光颜色等数据存储在中间缓存区，叫做G-buffer（G是geometry）。第二步，从G-buffer中读取信息，应用反射模型，计算出每个像素的最终颜色，即每个光源使用G-buffer数据将以一个2D后期处理的方式施加到最终图像上。
这种技术是我们避免了应用光照模型在最终不可见的片段上。对于屏幕上的每个像素，反射模型的计算只会发生一次。
这其实就是OpenGL中的帧缓存处理。

当使用Deferred Shading，Unity的渲染过程发生在两个passes：      

1. G-buffer Pass：物体使用 diffuse color, specular color, smoothness, world space normal, emission and depth 来渲染产生屏幕空间的缓存。   
2. Lighting pass：前面生成的缓存用于增加灯光到emission缓存。

不能使用deferred shading的物体，会在这个过程完成后使用forward渲染路径。

默认的g-buffer布局如下： 
       
- RT0, ARGB32 格式：Diffuse color (RGB), unused (A).
- RT1, ARGB32 格式: Specular color (RGB), roughness (A).
- RT2, ARGB2101010(30BIT) 格式: World space normal (RGB), unused (A).
- RT3, ARGB32 (non-HDR) 或 ARGBHalf (HDR) 格式: Emission + lighting + lightmaps + reflection probes buffer.
- Depth+Stencil buffer.

所以默认g-buffer布局是160位每像素（非HDR）或192位每像素（HDR）。

Emission+lighting（RT3）是对数编码的，在非相机HDR时，相比通常使用的ARGB32 texture会提供更大的动态范围。
注意当摄像机使用HDR 渲染，将没有为 Emission+lighting buffer (RT3) 创建的单独渲染目标，反而相机将要渲染的目标会使用RT3。

- G-Buffer Pass     
g-buffer pass渲染每个物体一次，Diffuse 和 specular 颜色, surface smoothness, world space normal, emission+ambient+reflections+lightmaps 会渲染进g-buffer贴图。g-buffer texture 是设置为全局shader属性，便于之后shader获取。

- Lighting Pass     
    Lighting Pass根据g-buffer和depth计算光照。光照是在屏幕空间计算的，所以它处理的时间是和场景复杂度无关的。灯光是添加到emission buffer。

    不穿过相机近平面的点光和聚光就像3D图形一样根据z缓冲测试被渲染。这会使部分或全部被堵住的点光/聚光渲染消耗很低。直光和穿越了相机近平面的点/聚光就会被渲染为全屏四边形。

    如果灯光开启了阴影，那么它们也会被渲染并应用在这个pass。阴影并不是“免费”的，阴影投射器也需要被渲染而且需要使用更复杂的shader。

    唯一可用的灯光模型就是Standard。如果你想用不同的模型，你可以修改 lighting pass shader，通过把修改过的内建Internal-DeferredShading.shader文件放到Resources文件夹。然后到Edit->Project Settings->Graphics窗口，更改Deferred到Custom Shader。


---			
> 参考：<br>
[Deferred Shading Rendering Path](http://docs.unity3d.com/Manual/RenderTech-DeferredShading.html)