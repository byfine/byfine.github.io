---
layout: post
title: Shader：Fixed function shader 简介
description: "Fixed function shader 简介"
modified: 2015-11-20
tags: [Shader]
---
## 一个顶点光照着色器的例子

    Shader "Custom/FF1" {
        Properties {
            _Color ("Main Color", Color) = (1,1,1,0)
            _Ambient ("Ambient", Color) = (0.5,0.5,0.5,0)
            _SpecColor ("Spec Color", Color) = (1,1,1,1)
            _Emission ("Emmisive Color", Color) = (0,0,0,0)
            _Shininess ("Shininess", Range (0.01, 1)) = 0.7
            _MainTex ("Base (RGB)", 2D) = "white" {}
        }
        SubShader {
            Pass {
                Material {
                    Diffuse [_Color]
                    Ambient [_Ambient]
                    Shininess [_Shininess]
                    Specular [_SpecColor]
                    Emission [_Emission]
                }
                Lighting On
                SeparateSpecular On
                SetTexture [_MainTex] {
                    Combine texture * primary DOUBLE
                }
            }
        }
    }

*首先要注意，shader不区分大小写。*

首先第一行是shader的名称，采用分级方式。定义好后可通过Shader.Find来找到对应名称的shader。

接下来是属性(Properties)。其中包含许多属性。前一部分是属性名，括号内字符串是显示的名字，后面跟着属性的类型。最后等号后面是默认值。
属性的类型有许多种包括数字（Float，Int），滑动条（Range）， 颜色（Color），Vector， 贴图（2D、Rect、3D、Cube）等。

下面是一个SubShader。每个Subshader都由一个或多个Passes组成，一个Pass块引起物体的一次渲染。所以至少要有一个Pass。
我们的Pass中有一个Material块，绑定了属性值到材质设置。

再下面是一些操作命令。

## 属性解释

- Color    
物体的纯色
- Diffuse Color   
漫反射颜色。这是物体的基本颜色。
- Ambient Color   
环境色颜色。物体在被RenderSettings里面设置的环境光照射时的颜色。
- Specular Color   
物体反射高光的颜色。
- Shininess Number   
高光的锐利度，在0到1之间，0时高光巨大，1时高光尖锐。
- Emission Color   
自发光，当物体没有被任何光线照射时的颜色。

- Lighting On | Off   
顶点光照的开关
- SeparateSpecular On | Off  
控制顶点光照的高光开关

## Combine texture 
SetTexture 读入材质内容，Combine将primary(前面光照计算的颜色)与材质纹理合并，DOUBLE表示乘以2以让颜色更亮。

SetTexture 可以不只有一个（支持多少个由硬件决定，一般两个就够了）。将上面例子改为

    .......

    Properties {
	......
	_MainTex ("Base (RGB)", 2D) = "white" {}
        _SecondTex ("Second (RGB)", 2D) = "white" {}
    }

	......

    SetTexture [_MainTex] {
        Combine texture * primary DOUBLE
    }
    SetTexture [_SecondTex] {
        Combine texture * previous DOUBLE
    }

第二个SetTexture 中的previous表示上一次SetTexture 计算的结果。从而可以混合两张纹理。

其他更为详细的命令，将在后面总结。

---
> 参考：
[Unity Manual - Shaders: ShaderLab & Fixed Function shaders](http://docs.unity3d.com/Manual/ShaderTut1.html)
