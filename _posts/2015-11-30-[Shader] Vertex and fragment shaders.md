---
layout: post
title: Shader：Vertex and Fragment shaders
description: "vertex and Fragment shaders 简介"
modified: 2015-11-30
tags: [Shader]
---

shader程序使用Cg/HLSL语言编写，为嵌入在pass内的代码片段，一般如下：

    Pass {
        // ... the usual pass state setup ...
        
        CGPROGRAM
        // compilation directives for this snippet, e.g.:
        #pragma vertex vert
        #pragma fragment frag
        
        // the Cg/HLSL code itself
        
        ENDCG
        // ... the rest of pass setup ...
    }

###Cg/HLSL片段

Cg/HLSL片段写在 CGPROGRAM 和 ENDCG 之间。   
在片段开始可以通过提供的 #pragma 命令声明编译指令。Unity认可的指令有以下这些：

- \#pragma vertex name - 以name为名字的 vertex shader 函数。
- \#pragma fragment name - 以name为名字的 fragment shader 函数。
- \#pragma geometry name - 以name为名字的 DX10 geometry shader 函数。使用这个指令，会默认打开 #pragma target 4.0。
- \#pragma hull name - 以name为名字的 DX11 hull shader 函数。使用这个指令，会默认打开 #pragma target 5.0。
- \#pragma domain name - 以name为名字的 DX11 domain sahder 函数。使用这个指令，会默认打开 #pragma target 5.0

其他编译指令：

- \#pragma target name - 指定编译的 shader target。
- \#pragma only_renderers space separated names - 只对指定的渲染器编译。
- \#pragma exclude_renderers space separated names - 不编译指定的渲染器。
- \#pragma multi_compile …_ - 用于multiple shader variants。
- \#pragma enable_d3d11_debug_symbols - 对DirectX 11的shader编译生成debug信息，允许你使用VS Graphics debugger 调试shader。

每个代码片段至少要包含一个顶点程序和一个片段程序。因此 \#pragma vertex 和 \#pragma fragment 指令是必须的。

###顶点和片断程序
当你使用顶点和片断程序(即可编程管线)时，显卡的大部分硬编码(固定功能管线)功能将关闭。
例如，使用一个顶点程序完全可以做到关闭标准的3D变换，灯光和纹理坐标的功能。类似的，使用一个片段程序可以替换任何纹理混合模式，而这些纹理混合模式都在在SetTexture命令中有定义，因此SetTexture命令是不需要的。

编写顶点/片断程序需要对3D转换、照明和坐标空间有透彻的了解。因为你要自己写出像OpenGL实现的固定功能一样的效果。另外，还可以实现内置功能以外自己需要的功能。

###在ShaderLab中使用Cg/HLSL
ShaderLab中的shader通常使用Cg/HLSL编写。Cg和DX9的HLSL几乎一样，所以可以交换使用。

Shader代码写在“Cg/HLSL代码段“中。代码段会被编译为低级着色器集合，并且最终的着色器是包含在你的游戏数据文件内的。当你在Project View选中一个shader文件，Inspetor窗口会有个按钮可以查看编译后的着色器代码，可以帮助调试。Unity会自动编译Cg片段到相关平台。因为Cg/HLSL代码是通过Unity editor编译的，所以不能在运行时创建shader。


下面的例子演示了一个完整Cg程序的着色器，它的结果是颜色随着法线而变化：

    Shader "Tutorial/Display Normals" {
        SubShader {
            Pass {

                CGPROGRAM

                #pragma vertex vert
                #pragma fragment frag
                #include "UnityCG.cginc"

                struct v2f {
                    float4 pos : SV_POSITION;
                    fixed3 color : COLOR0;
                };

                v2f vert (appdata_base v)
                {
                    v2f o;
                    o.pos = mul (UNITY_MATRIX_MVP, v.vertex);
                    o.color = v.normal * 0.5 + 0.5;
                    return o;
                }

                fixed4 frag (v2f i) : SV_Target
                {
                    return fixed4 (i.color, 1);
                }
                ENDCG

            }
        }
    }
    
   
这个着色器没有属性，有一个SubShader，包含一个pass。我们来分析一下这段Cg代码：

    CGPROGRAM
    #pragma vertex vert
    #pragma fragment frag
    // ...
    ENDCG   
    
整个Cg片段写在CGPROGRAM与ENDCG关键字中间。开始编译的指令是 #pragma 给出，指定了顶点函数vert 和 片段函数frag。

    #include UnityCg.cginc

这是普通的Cg代码，包含一个内置的文件。
UnityCg.cginc文件包含了常用的声明，所以可以使着色器简短。这里我们使用文件中定义的 appdata_base 结构体。

接下来我们定义了一个"vertex to fragment"结构，即v2f。定义了从顶点程序传递到片段程序的数据。我们传递了位置和颜色参数。颜色会在顶点程序中计算然后在片段程序中输出。

在顶点函数-vert中，我们计算顶点位置，然后将输入的法线输出为颜色：
    
    o.color = v.normal * 0.5 + 0.5;

法线的范围是[-1, 1]，而颜色范围是[0, 1]。所以通过上面计算调整。接下来定义一个片段程序，它只返回计算好的颜色：

    return fixed4 (i.color, 1);
    
到此，我们就分析完了这个着色器。虽然只是个简单的着色器，但很方便查看网格的法线。

###在Cg/HLSL代码中使用shader属性

####例子介绍
在Cg代码中使用着色器属性(shader properties)，你必须定义一个变量，变量的的名字和类型要与它相匹配。
这里有个完整的shader例子，用来显示通过颜色调整的贴图：

    Shader "Tutorial/Textured Colored" {
        Properties {
            _Color ("Main Color", Color) = (1,1,1,0.5)
            _MainTex ("Texture", 2D) = "white" { }
        }
        SubShader {
            Pass {

            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            #include "UnityCG.cginc"

            fixed4 _Color;
            sampler2D _MainTex;

            struct v2f {
                float4 pos : SV_POSITION;
                float2 uv : TEXCOORD0;
            };

            float4 _MainTex_ST;

            v2f vert (appdata_base v)
            {
                v2f o;
                o.pos = mul (UNITY_MATRIX_MVP, v.vertex);
                o.uv = TRANSFORM_TEX (v.texcoord, _MainTex);
                return o;
            }

            fixed4 frag (v2f i) : SV_Target
            {
                fixed4 texcol = tex2D (_MainTex, i.uv);
                return texcol * _Color;
            }
            ENDCG

            }
        }
    }
    
这个shader和上面的shader结构相同。在这个例子里，我们定义了两个属性，名字为 _Color 和 _MainTex。于是在Cg/HLSL代码中，我们也相应的定义了属性：
    
    fixed4 _Color;
    sampler2D _MainTex;

这里的顶点程序使用UnityCG.cginc里的TRANSFORM_TEX，用来保证纹理(texture)正确的缩放和偏移。片断(fragment)程序只是对纹理(texture)进行采样然后乘以颜色值。

####属性类型：

对于下面的shader属性：

    _MyColor ("Some Color", Color) = (1,1,1,1) 
    _MyVector ("Some Vector", Vector) = (0,0,0,0) 
    _MyFloat ("My float", Float) = 0.5 
    _MyTexture ("Texture", 2D) = "white" {} 
    _MyCubemap ("Cubemap", CUBE) = "" {} 

Cg/HLSL代码中为了获取，需要定义为：

    fixed4 _MyColor; //对于颜色通常低精度类型就足够了
    float4 _MyVector;
    float _MyFloat; 
    sampler2D _MyTexture;
    samplerCUBE _MyCubemap;
    
Cg/HLSL也可以接受uniform，但是没有必要：
    
    uniform float4 _MyColor;
    
ShaderLab的映射到Cg/HLSL的属性类型：

- Color 和 Vector 属性映射为 float4, half4 或 fixed4 变量。
- Range 和 Float 属性映射为 float, half 或 fixed 变量。
- Texture 属性。对于2Dtextures 为 sampler2D; Cubemaps 为 samplerCUBE; 3D textures 为 sampler3D.

####特殊的贴图属性：

#####Texture tiling & offset：

对于贴图材质通常有Tiling 和 Offset数据块。在shader中，这个信息是通过一个名为{TextureName}_ST的float4属性传递： 
   
x 是 X tiling； y 是 Y tiling； z 是 X offset； w 是 Y offset。

比如，一个贴图名为 _MainTex，那么它的tilling信息为一个_MainTex_ST的vector。

#####Texture size：

{TextureName}_TexelSize - 一个float4属性包含了贴图的size信息：

x 是 1.0/width； y 是 1.0/height； z 是 width； w 是 height

#####Texture HDR parameters：
{TextureName}_HDR - 一个float4属性，包含了怎样解码潜在的HDR贴图信息。


---			
> 参考：<br>
[Writing vertex and fragment shaders](http://docs.unity3d.com/Manual/SL-ShaderPrograms.html)<br>
[Shaders: Vertex and Fragment Programs](http://docs.unity3d.com/Manual/ShaderTut2.html)<br>
[Accessing shader properties in Cg/HLSL](http://docs.unity3d.com/Manual/SL-PropertiesInPrograms.html)