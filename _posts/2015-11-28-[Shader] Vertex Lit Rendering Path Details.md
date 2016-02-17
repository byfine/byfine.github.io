---
layout: post
title: Shader：Vertex Lit Rendering Path Details
description: "Vertex Lit Rendering Path 简介"
modified: 2015-11-28
tags: [Shader, Unity3D]
---

### 简介：
Vertex Lit path通常在一个pass里渲染每个物体，在物体顶点计算所有灯光的光照。每个顶点的光照计算出后，会对灯光进行插值，所以多边形效果很明显。

这是最快的渲染路径，并支持硬件最广泛。

因为所有光照都是计算在顶点的，这个通道并不支持逐像素效果：阴影，法线贴图，light cookies，高度细节贴图等都不支持。

---			
> 参考：<br>
[Vertex Lit Rendering Path Details](http://docs.unity3d.com/Manual/RenderTech-VertexLit.html)<br>