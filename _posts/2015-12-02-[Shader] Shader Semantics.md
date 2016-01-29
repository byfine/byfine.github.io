---
layout: post
title: Shader：Shader 语义（Semantics）
description: "Shader语义简介"
modified: 2015-12-02
tags: [Shader]
---

当编写Cg/HLSL程序，输入和输出变量需要通过“语义”来表达它们的意图。这是HLSL语言中的概念，可以查看[Semantics documentation on MSDN](https://msdn.microsoft.com/en-us/library/windows/desktop/bb509647.aspx)。

###顶点着色器输入语义：
主顶点函数（#pragma vertex 指定的函数）对于所有输入参数都需要有语义。这些是与单一的Mesh元素对应的，比如顶点位置、法线、纹理坐标等。

这里有一个顶点shader的例子，以顶点位置和纹理坐标为输入。每个像素都显示为纹理坐标表示的颜色。

    Shader "Unlit/Show UVs"
    {
        SubShader
        {
            Pass
            {
                CGPROGRAM
                #pragma vertex vert
                #pragma fragment frag

                struct v2f {
                    float2 uv : TEXCOORD0;
                    float4 pos : SV_POSITION;
                };

                v2f vert (
                    float4 vertex : POSITION, // vertex position input
                    float2 uv : TEXCOORD0 // first texture coordinate input
                    )
                {
                    v2f o;
                    o.pos = mul(UNITY_MATRIX_MVP, vertex);
                    o.uv = uv;
                    return o;
                }
                
                fixed4 frag (v2f i) : SV_Target
                {
                    return fixed4(i.uv, 0, 0);
                }
                ENDCG
            }
        }
    }
    
不同于分别拼出所有的输入，可以使用结构体来声明，并对每个结构体每个单独的成员指定语义。

###片段着色器输出语义
一般片断着色器是输出一个颜色，并且有一个SV_Target语义。比如上面的例子中：

    fixed4 frag (v2f i) : SV_Target

这个frag函数有一个fixed4类型的返回值，它只返回一个值由SV_Target语义示意。

也可以输出一个结构体，上面的例子可以改写成这样：
    
    struct fragOutput {
        fixed4 color : SV_Target;
    };            
    fragOutput frag (v2f i)
    {
        fragOutput o;
        o.color = fixed4(i.uv, 0, 0);
        return o;
    }

返回结构体通常用于返回值不是一个单一的颜色。片断着色器还支持的其他语义有：

**Multiple Render Targets: SV_TargetN**     
SV_Target1, SV_Target2, … - 由shader写的额外颜色。用于一次渲染到多于一个的渲染目标时。SV_Target0 和 SV_Target 一样。

**Pixel Shader Depth Output: SV_Depth**     
SV_Depth - 覆盖深度缓存值。通常片段着色器不覆盖Z缓存，而是来自常规的三角形光栅化。然而为了实现某些效果，可以输出每个像素用户自定义的深度值。

注意对许多GPU为了深度缓存优化，这个是关闭的。所以除非必要不要重写Z缓存。SV_Depth的消耗与具体的GPU架构有关，但一般与alpha testing（HLSL内建的clip()函数）基本相当。

输出的深度值需要是一个float。


###顶点着色器输出和片段着色器输入
一个顶点着色器需要输出每个顶点的最终”裁剪空间“位置，以便GPU知道渲染的位置以及光栅化深度。这个输出需要有SV_POSITION语义，并且是float4类型。

顶点着色器输出的值，会在渲染的三角形面之间进行插值，然后每个像素的这个值会作为输入传递到片段着色器。

许多现代GPU并不关心这些值有什么语义，然而许多老的系统（SM2.0或是DX9）还是对语义有特殊要求：

- TEXCOORD0, TEXCOORD1, … ， 用于指示任意精度数据 - 纹理坐标、位置等。
- COLOR0 和 COLOR1，用于低精度0-1范围数据（颜色）。

为了更好的跨平台支持，通常把顶点输出和片段输入标记为 TEXCOORDn 语义。

###其他特殊语义

####屏幕空间像素位置: VPOS
一个片段着色器可以接收被渲染的像素的位置。这个特性只支持SM2.0，所以需要声明 #pragma target 3.0 编译指令。

在不同平台，像素坐标的类型是不同的，所以为了方便使用 UNITY_VPOS_TYPE 类型。

另外，使用了像素坐标语义，会使在一个v2f结构体内同时获取裁剪空间坐标（SV_POSITION）和VPOS变得困难。所以顶点着色器会通过一个单独的输出变量，来输出裁剪空间坐标。请看一个例子：

    Shader "Unlit/Screen Position"
    {
        Properties
        {
            _MainTex ("Texture", 2D) = "white" {}
        }
        SubShader
        {
            Pass
            {
                CGPROGRAM
                #pragma vertex vert
                #pragma fragment frag
                #pragma target 3.0

                // note: no SV_POSITION in this struct
                struct v2f {
                    float2 uv : TEXCOORD0;
                };

                v2f vert (
                    float4 vertex : POSITION, // vertex position input
                    float2 uv : TEXCOORD0, // texture coordinate input
                    out float4 outpos : SV_POSITION // clip space position output
                    )
                {
                    v2f o;
                    o.uv = uv;
                    outpos = mul(UNITY_MATRIX_MVP, vertex);
                    return o;
                }

                sampler2D _MainTex;

                fixed4 frag (v2f i, UNITY_VPOS_TYPE screenPos : VPOS) : SV_Target
                {
                    // screenPos.xy will contain pixel integer coordinates.
                    // use them to implement a checkerboard pattern that skips rendering
                    // 4x4 blocks of pixels

                    // checker value will be negative for 4x4 blocks of pixels
                    // in a checkerboard pattern
                    screenPos.xy = floor(screenPos.xy * 0.25) * 0.5;
                    float checker = -frac(screenPos.r + screenPos.g);

                    // clip HLSL instruction stops rendering a pixel if value is negative
                    clip(checker);

                    // for pixels that were kept, read the texture and output it
                    fixed4 c = tex2D (_MainTex, i.uv);
                    return c;
                }
                ENDCG
            }
        }
    }    

####面的方向: VFACE
片段着色器可以接收一个变量，指示渲染的表面是面向摄像机还是背向摄像机。这对于需要渲染双面的几何体很有效。VFACE语义的输入值，对正面会包含一个正值，反面包含一个负值。

只支持SM3.0。

    Shader "Unlit/Face Orientation"
    {
        Properties
        {
            _ColorFront ("Front Color", Color) = (1,0.7,0.7,1)
            _ColorBack ("Back Color", Color) = (0.7,1,0.7,1)
        }
        SubShader
        {
            Pass
            {
                Cull Off // turn off backface culling

                CGPROGRAM
                #pragma vertex vert
                #pragma fragment frag
                #pragma target 3.0

                float4 vert (float4 vertex : POSITION) : SV_POSITION
                {
                    return mul(UNITY_MATRIX_MVP, vertex);
                }

                fixed4 _ColorFront;
                fixed4 _ColorBack;

                fixed4 frag (fixed facing : VFACE) : SV_Target
                {
                    // VFACE input positive for frontbaces,
                    // negative for backfaces. Output one
                    // of the two colors depending on that.
                    return facing > 0 ? _ColorFront : _ColorBack;
                }
                ENDCG
            }
        }
    }

首先使用Cull Off关闭背面剔除（默认是剔除背面的）。然后对正反面设置不同的颜色。

####顶点ID：SV_VertexID
顶点着色器可以接收一个变量，它包含一个unsigned int类型的顶点编号。通常用于你希望从贴图或ComputeBuffers获取额外的每顶点信息。

只支持DX10（SM4.0）或 GLCore / OpenGL ES 3，所以需要编译指令：#pragma target es3.0。

    Shader "Unlit/VertexID"
    {
        SubShader
        {
            Pass
            {
                CGPROGRAM
                #pragma vertex vert
                #pragma fragment frag
                #pragma target es3.0

                struct v2f {
                    fixed4 color : TEXCOORD0;
                    float4 pos : SV_POSITION;
                };

                v2f vert (
                    float4 vertex : POSITION, // vertex position input
                    uint vid : SV_VertexID // vertex ID, needs to be uint
                    )
                {
                    v2f o;
                    o.pos = mul(UNITY_MATRIX_MVP, vertex);
                    // output funky colors based on vertex ID
                    float f = (float)vid;
                    o.color = half4(sin(f/10),sin(f/100),sin(f/1000),0) * 0.5 + 0.5;
                    return o;
                }

                fixed4 frag (v2f i) : SV_Target
                {
                    return i.color;
                }
                ENDCG
            }
        }
    }



---			
> 参考：<br>
[Shader Semantics](http://docs.unity3d.com/Manual/SL-ShaderSemantics.html)