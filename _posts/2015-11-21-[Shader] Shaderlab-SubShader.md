---
layout: post
title: Shader：ShaderLab语法-SubShader
description: "ShaderLab语法-SubShader"
modified: 2015-11-21
tags: [Shader]
---

每个shader都由一列SubShader构成。真正用于呈现渲染物体的内容是在SubShader中实现的。Unity在实际运行时，会根据硬件情况从上到下选择最优的一个SubShader来执行。

## 语法

    Subshader { 
        [Tags] 
        [CommonState] 
        Passdef [Passdef ...] 
    }

## 细节
每个SubShader都定义了一系列passes，并可对所有passes可选的设置任一状态。另外，还可以指定[Tags]。
当Unity选定了一个SubShader，它会对定义的每一个pass都渲染一遍。有时在某些硬件某些复杂的渲染效果必须在多个pass中完成。

每个pass都可以被定义成 [regular Pass](http://docs.unity3d.com/Manual/SL-Pass.html),  [Use Pass](http://docs.unity3d.com/Manual/SL-UsePass.html) or [Grab Pass](http://docs.unity3d.com/Manual/SL-GrabPass.html).


## Pass简介
一个pass块引起一次物体的几何学渲染。

- 语法        
    Pass { [Name and Tags] [RenderSetup] }
    
- Name and tags      
一个pass块可以定义它的名字和任意数量的Tags。
内置的Tags都是针对渲染路径的，告诉渲染引擎这个pass块应该在什么路径下被渲染。
Name一般被用来引用Pass块。pass名字都是大写。

## UsePass
UsePass命令可以使用另一个shader中的pass块，从而减少重复劳动。

格式：UsePass "Shader/Name"

若要UsePass能工作，所引用的pass块必须定义了名字。

## GrabPass
GrabPass是一种特殊的pass类型，它可以捕获物体所在位置的屏幕内容并写入到一个纹理中。从而用于后续通道完成高级图像效果。

语法： 

- GrabPass{}，能捕获当前屏幕的内容到一个纹理中，纹理能在后续通道中通过 _GrabTexture 进行访问。注意：这种形式的捕获通道对于每个使用它的物体将进行昂贵的屏幕捕捉操作。
- GrabPass { "TextureName" }，能捕获屏幕内容到一个纹理中，但对于第一个给定纹理名的物体，只会在每帧中处理一次。在后面的通道中可以通过纹理名获取纹理。当你在一个场景中有多个物体使用grab pass，这是更高效的方法。

另外, GrabPass 能使用 [Name](http://docs.unity3d.com/Manual/SL-Name.html) 和 [Tags](http://docs.unity3d.com/Manual/SL-PassTags.html) 命令。


## Tags
SubShader使用Tags告诉渲染引擎期望何时和如何渲染对象。

#### 语法    
字符串的键值对：

    Tags { 
        "TagName1" = "Value1" 
        "TagName2" = "Value2" 
    }

#### 细节    
在一个SubShader中标签被用于决定渲染次序和其他SubShader的参数。
注意接下来的标签只能置于SubShader块而不是Pass块中。
除了可被Unity认可的内置标签，你还可以使用自己的标签并通过[Material.GetTag](http://docs.unity3d.com/ScriptReference/Material.GetTag.html)函数查询。
  
- Rendering Order - Queue tag    
    使用 Queue 标签决定对象被渲染的次序。比如确保透明（Transparent）物体在不透明（opaque）物体之后绘制。
    有四种预定义队列，但可以在其之间定义更多队列：
    
    1. Background: 在所有队列之前被渲染，被用于渲染天空盒之类。
    2. Geometry *(default)*: 默认的，被用于大多数对象。不透明的几何体使用这个队列。
    3. AlphaTest: alpha测试，这是分离自Geometry的队列，因为对于alpha-tested物体，在所有固体渲染后再渲染更有效率。
    4. Transparent: 透明，在Geometry队列之后被渲染，采用由后到前的次序。
    5. Overlay: 覆盖，用于实现叠加效果。任何需要最后渲染的对象应该放置在此处。（如镜头光晕）。 
    
    每一个队列都代表一个整数索引值。Background-1000，Geometry-2000，AlphaTest-2450，Transparent-3000，Overlay-4000。可以自定义一个队列：
    
    Tags { "Queue" = "Geometry+1" }

    该队列的索引值为2001，这会使物体在所有不透明的对象之后但却在所有透明物体前被渲染。

- RenderType    
    将shader分类成预定义的组。

- DisableBatching   
    一些shader在[Draw Call Batching](http://docs.unity3d.com/Manual/DrawCallBatching.html) 模式下不能工作，这个指令来决定是否禁用它。
    “True”：禁用。“False”:不禁用（默认）。“LODFading”:当LOD Fading开启时禁用，一般用于树木。

- ForceNoShadowCasting  
    “True”：不使用cast shadows。

- IgnoreProjector   
    “True”：物体不会受 [Projectors](http://docs.unity3d.com/Manual/class-Projector.html)影响。

- CanUseSpriteAtlas     
    “False”：如果这个shader是指sprites，将无法打包成图集。 ([Sprite Packer](http://docs.unity3d.com/Manual/SpritePacker.html))

- PreviewType   
    指示material inspector预览怎样显示材质。默认显示是spheres，也可设置为“Plane”（2D显示），或者"Skybox"（天空球显示）。

---
pass中也可以使用Tags

---
> 参考：
[Unity Manual - ShaderLab: SubShader](http://docs.unity3d.com/Manual/SL-SubShader.html)
