---
layout: post
title: Shader：Legacy Deferred Lighting Rendering Path (light prepass)
description: "Legacy Deferred Lighting Rendering Path (light prepass) 简介"
modified: 2015-11-27
tags: [Shader, Unity3D]
---

### 注意

1. Deferred Lighting在Unity5.0后被认为是遗留功能，因为它不支持一些渲染特性（比如Standard shader, reflection probes）。新的项目推荐使用Deferred Shading渲染通道代替。

2. 当相机是Orthographic投影，并不支持Deferred模式。会被处理为Forward rendering模式。

### 预览
当时用Deferred Lighting，影响物体的光源没有数量限制。所有灯光都是逐像素计算，意味着都可以正确的与法线贴图交互等等。另外所有灯光都可以有cookies和阴影。

Deferred lighting的优势是处理消耗和灯光照亮的像素数成正比。这只与灯光体积有关而与它照到的物体数量无关。因此保持较小的光体积可以提高性能。因为它是逐像素计算光照，所以不像顶点着色多边形明显，不真实。

但缺点是，deferred shading并不真正支持抗锯齿，也不能处理半透明物体（这是在forward rendering中处理）。也不支持Mesh Renderer的Receive Shadows标志，而且只能使用4个culling masks。

### 需求
需要Shader Model 3.0及以上，支持Depth render textures和two-sided stencil buffers，2004年后的大部分的pc都支持。移动平台上，所有兼容OpenGL ES 3.0的GPU都支持deferred lighting。

### 性能考量
灯光的渲染开销与灯光照射的像素数有关而与场景的复杂度无关。所以小的点光或聚光开销很少，而如果它们被场景物体全部或部分挡住，开销会更少。

当然，使用阴影的灯光更加消耗性能。在deferred lighting中，对每个阴影投射灯光，阴影投射的物体仍需要被渲染一次或多次。此外，应用了阴影的灯光shader拥有更多渲染开销。

### 实现细节
当时用deferred lighting，渲染过程发生在三个通道：

- Bass pass：渲染生成屏幕空间缓冲信息：depth, normals, 和specular power.
- Lighting pass：使用上面计算的缓冲来计算灯光到另一个屏幕空间缓存。
- Final pass：物体再次被渲染。获取上面计算的光照和贴图再加上环境光/自发光等等混合在一起。
    
不能使用deferred shading的物体，会在这个过程完成后使用forward渲染路径。

- Base Pass     
    渲染每个物体一次。视图空间法线和specular power会被渲染为一个ARGB32的Render Texture，法线使用RGB通道，specular power使用A通道。如果平台允许Z缓存从贴图读取，那么深度值不会明确的渲染。如果深度不能从贴图读取，那么深度会使用shader replacement在额外的通道渲染。

    Base Pass的结果就是把场景信息和一张包含normal与specular power信息的Render Texture 填充在z buffer中。

- Lighting Pass     
    Lighting pass 根据depth, normals and specular power计算光照值。灯光在屏幕空间中计算，所以消耗时间与场景复杂度无关。灯光缓存是一个ARGB32 Render Texture，包含使用RGB通道的diffuse lighting和使用A通道的单色specular lighting。灯光值是用对数编码的，以提供相对于使用ARGB32 texture更大的动态范围。当摄像机开启了HDR rendering，那么灯光缓存会使用ARGBHalf格式，而且不会使用对数编码。
	不穿过相机近平面的点光和聚光就像3D图形一样根据z缓冲测试被渲染。穿过近平面的光也会想3D图形一样渲染，但是背面的面使用相反的深度测试。这会使部分或全部被堵住的点光/聚光渲染消耗很低。如果一个灯光既与相机近平面又与远平面相交，那么上面的优化方法就不能使用，灯光会被渲染为没有深度的紧密四边形。
	
	以上并不适用于方向光，方向光总会渲染为全屏四边形。
	
	如果一个灯光开启了阴影那么也会在这个pass渲染。
	
	唯一允许的灯光模型是Blinn-Phong。如果lighting pass shader想使用不同的光照模型，可以把修改过的内建Internal-PrePassLighting.shader文件放到Resources文件夹。然后到Edit->Project Settings->Graphics窗口，更改Deferred到Custom Shader。
	
- Final Pass        
	Final Pass产生最终的渲染贴图。获取灯光并混合贴图与额外的发散光来渲染所有物体。光照贴图也会在这个通道应用。相机离得近的地方进行实时光照，只烘焙 indirect lighting。相机离得远的地方，全部烘焙。

---			
> 参考：<br>
[Legacy Deferred Lighting Rendering Path](http://docs.unity3d.com/Manual/RenderTech-DeferredLighting.html)<br>