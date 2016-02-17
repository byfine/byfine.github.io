---
layout: post
title: Shader：ShaderLab语法-Properties
description: "ShaderLab语法-Properties"
modified: 2015-11-20
tags: [Shader]
---

如前所述，Unity中的所有Shaders都是用“ShaderLab”这种声明式语言（Declarative programming）编写的。真正的“shader code”写在同一shader文件的*CGPROGRAM* snippets内。*CGPROGRAM* snippets 是用通用的 HLSL/Cg 语言编写的。

#### Properties

Properties { Property [Property ...] }

#### 格式
_Name(“Display Name”,type) = defaultValue{options}
属性名（“显示名”，类型） = 默认值 {可选参数}

#### 数字和滑动条
**Float**、**Int**、**Range (min, max)**  
这些都使用默认值定义了数字属性。 Range格式会显示一个min到max的滑动条。

    name ("display name", Range (min, max)) = number
    name ("display name", Float) = number

#### Colors and Vectors
定义一个RGBA颜色，或者一个4D向量属性。

    name ("display name", Color) = (number,number,number,number)
    name ("display name", Vector) = (number,number,number,number)

#### Textures
定义一个[2D texture](http://docs.unity3d.com/Manual/class-TextureImporter.html), [cubemap](http://docs.unity3d.com/Manual/class-Cubemap.html) or [3D (volume)](http://docs.unity3d.com/Manual/class-Texture3D.html) 属性。

    name ("display name", 2D) = "defaulttexture" {}
    name ("display name", Cube) = "defaulttexture" {}
    name ("display name", 3D) = "defaulttexture" {}

- 2D:一张2的阶数大小的贴图。这张贴图将在采样后被转为对应基于模型UV的每个像素的颜色，最终显示出来。
- Cube: 即Cube map texture（立方体纹理），简单说就是6张有联系的2D贴图的组合，主要用来做反射效果（比如天空盒和动态反射），也会被转换为对应点的采样。
- 3D: Unity支持3D纹理，从着色器和脚本使用并创建。目前，3D纹理只能通过脚本创建。
    
#### 细节
每个属性都是根据**name**引用的（在Unity中一般以下划线_开始），而在编辑器界面显示的是**display name**，等号后面指定了默认值：

- 对于Range和Float其只是一个浮点数，color范围0~1
- Color和Vector是括号内的四个数
- 贴图（2D、Cube）默认值是一个空字符串，或者是内建的默认贴图：“*white*”, “*black*”, “*gray*” or “*bump*”.

之后在shader的fixed function部分，属性值可以通过方括号加属性名获取：[name]。
Properties块中的属性会被序列化成material数据，这对于在脚本控制值非常有用 (使用 [Material.SetFloat](http://docs.unity3d.com/ScriptReference/Material.SetFloat.html) 或类似函数).

#### Property attributes and drawers
对之前的属性，都可以使用方括号指定可选的修饰（attributes），下面是一些Unity可以认出的修饰，或是可以自己指定（[MaterialPropertyDrawer classes](http://docs.unity3d.com/ScriptReference/MaterialPropertyDrawer.html)） 以控制它们在material  inspector中如何被渲染。

- [HideInInspector] 不在inspector显示
- [NoScaleOffset] 对于texture属性，不会显示贴图的 tiling/offset字段。
- [Normal] 指示一个贴图是normal-map。
- [HDR] 指示一个贴图是高动态范围（high-dynamic range (HDR)）贴图。
- [PerRendererData] 指示一个贴图是来自[MaterialPropertyBlock](http://docs.unity3d.com/ScriptReference/MaterialPropertyBlock.html)格式的 per-renderer data 。

---
> 参考：
[Unity Manual - ShaderLab: Properties](http://docs.unity3d.com/Manual/SL-Properties.html)