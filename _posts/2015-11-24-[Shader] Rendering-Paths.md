---
layout: post
title: Shader：Unity Rendering Paths 简介
description: "Unity Rendering Paths 简介"
modified: 2015-11-24
tags: [Shader, Unity3D]
---

###Rendering Paths
Unity支持不同的渲染路径，你可以根据你的游戏内容和目标平台选择合适的使用。不同渲染路径有不同的表现特征，区别主要体现在灯光和阴影上。

Unity中可以在Player Setting设置渲染路径，也可以为每个摄像机单独设置。    
Unity共支持四种渲染路径：

- Deferred Shading      
这是灯光和阴影效果最真实的渲染路径，最适合有许多实时灯光的情况。需要一定水平的硬件支持。

- Forward Rendering     
这是传统的渲染路径，支持所有典型的图形特性（法线贴图，像素光照，阴影等）。然而，在默认设置下，只有少数最亮的灯光可以使用逐像素光照模式，其他灯光对每个物体使用逐顶点计算。

- Legacy Deferred       
Legacy Deferred (light prepass)与Deferred类似，根据权衡使用了不同技术。不支持Unity5的基于物理标准shader。

- Legacy Vertex Lit     
这是最低真实度的渲染路径，不支持实时阴影。是 Forward rendering path 的子集。
	
注意，当摄像机为Orthographic投影，不能使用 Deferred rendering 。如果使用Orthographic，总会使用Forward rendering。

![]({{ site.url }}/images/post/Shader/Rendering-Paths.png)

###Unity's Rendering Pipeline
shaders既定义了一个物体自身如何显示（它的材质属性）也定义了它如何反应灯光。因为灯光计算必须在shader中构建，而且有许多不同的灯光和阴影类型，所以写出正好合适的shader是复杂的工作。为了使这个过程更简单，Unity拥有Surface shader，所有的灯光、阴影、光照贴图都会自动的处理为forward或是deferred。

- Rendering Paths   
灯光如何应用和哪个pass被使用，是依据Rendering path决定的。每个pass通过pass tag传达它的灯光类型。

    - Forward Rendering：ForwardBase 和 ForwardAdd pass会被使用。
    - Deferred Shading：Deferred pass被使用。
    - legacy Deferred Lighting：PrepassBase 和 PrepassFinal passes 被使用。
    - legacy Vertex Lit：Vertex, VertexLMRGBM 和 VertexLM passes 被使用。
    - 如果不包括上述任何情况，为了渲染阴影和深度贴图，ShadowCaster pass 被使用。

- Forward Rendering path    
ForwardBase pass 一次渲染ambient, lightmaps, 主要的 directional light 和不重要的(vertex/SH)灯光。ForwardAdd pass 用与额外的逐像素灯光，每个被这个灯光照明的物体调用一次。
如果forward rendering 使用了，但并没有合适的pass，物体会渲染为和使用Vertex Lit path一样。

- Deferred Shading path     
Deferred pass 为灯光渲染所有需要的信（对于内建shader：diffuse color, specular color, smoothness, world space normal, emission）息。还会添加光照贴图，reflection probes 和 ambient lighting 到emission通道。

- Legacy Deferred Lighting path     
PrepassBase pass 渲染法线和高光指数；PrepassFinal pass 渲染最终颜色，通过混合贴图、灯光和emissive材质属性。场景内的灯光分别的在屏幕空间完成。

- Legacy Vertex Lit Rendering path      
因为顶点光照多用于不支持可编程shader的平台，Unity不能内部创建多个shader变体来处理 lightmapped vs. non-lightmapped 的情况。所以为了处理 lightmapped 和 non-lightmapped 物体，必须要编写多个passes。

    - Vertex pass： 用于non-lightmapped 物体。一次渲染所有灯光，使用固定函数光照模型（Blinn-Phong）。
    - VertexLMRGBM pass： 用于 lightmapped 物体，当光照贴图是RGBM编码（PC和主机）。没有实时光照应用，pass用于混合贴图和光照贴图。
    - VertexLMM pass： 用于 lightmapped 物体，当光照贴图是double-LDR编码（移动平台）。没有实时光照应用，pass用于混合贴图和光照贴图。

---			
> 参考：<br>
[Rendering Paths](http://docs.unity3d.com/Manual/RenderingPaths.html)<br>
[Unity's Rendering Pipeline](http://docs.unity3d.com/Manual/SL-RenderPipeline.html)