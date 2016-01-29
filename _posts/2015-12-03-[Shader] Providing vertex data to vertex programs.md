---
layout: post
title: Shader：向顶点程序传递顶点数据
description: "Providing vertex data to vertex programs"
modified: 2015-12-03
tags: [Shader]
---

###顶点数据

对Cg/HLSL顶点程序，Mesh的顶点数据是作为输入传入到顶点着色器函数的。每个输入需要指定语义，比如POSITION表示顶点位置，NORMAL表示顶点法线等。

通常使用结构体输入顶点数据，而不是一个个分开的。几个常用的顶点数据都定义在 UnityCG.cginc 文件，通常使用它们就足够了。这些结构体包括：

- appdata_base: position, normal and one texture coordinate.
- appdata_tan: position, tangent, normal and one texture coordinate.
- appdata_full: position, tangent, normal, four texture coordinates and color.

如果你想获取不同的顶点数据，你需要自己声明结构体，或者添加输入参数。顶点数据通过语义区分，并且来自以下列表：

- POSITION ： 顶点坐标，通常是 float3 / float4.
- NORMAL ： 顶点法线,通常是 float3.
- TEXCOORD0 ： 第一个 UV coordinate, 通常是 float2..float4.
- TEXCOORD1 .. TEXCOORD3 ：  第二到第四个 UV coordinates.
- TANGENT ： 切向量 (用于法线映射), 通常是float4.
- COLOR ： 每个顶点的颜色, 通常是 float4.

当mesh数据包含的少于需要的输入数据，多余的会填充为0，除了 .w 默认为1。比如，通常纹理坐标是 2D 向量，如果定义了float4类型的TEXCOORD0输入，那么获取的值是：(x,y,0,1)。

###例子

####可视化UVs
下面的例子使用顶点位置和第一个纹理坐标作为顶点输入（在appdata内定义）。这个shader对于调试UV坐标非常有用。UV坐标显示为红色和绿色，而超过0-1范围的坐标显示为额外的蓝色。

    Shader "Debug/UV 1" {
        SubShader {
            Pass {
                CGPROGRAM
                #pragma vertex vert
                #pragma fragment frag

                // vertex input: position, UV
                struct appdata {
                    float4 vertex : POSITION;
                    float4 texcoord : TEXCOORD0;
                };

                struct v2f {
                    float4 pos : SV_POSITION;
                    float4 uv : TEXCOORD0;
                };
                
                v2f vert (appdata v) {
                    v2f o;
                    o.pos = mul( UNITY_MATRIX_MVP, v.vertex );
                    o.uv = float4( v.texcoord.xy, 0, 0 );
                    return o;
                }
                
                half4 frag( v2f i ) : SV_Target {
                    half4 c = frac( i.uv );
                    if (any(saturate(i.uv) - i.uv))
                        c.b = 0.5;
                    return c;
                }
                ENDCG
            }
        }
    }

####可视化顶点颜色
这个shader使用顶点位置和每个顶点颜色作为输入。

    Shader "Debug/Vertex color" {
        SubShader {
            Pass {
                CGPROGRAM
                #pragma vertex vert
                #pragma fragment frag

                // vertex input: position, color
                struct appdata {
                    float4 vertex : POSITION;
                    fixed4 color : COLOR;
                };

                struct v2f {
                    float4 pos : SV_POSITION;
                    fixed4 color : COLOR;
                };
                
                v2f vert (appdata v) {
                    v2f o;
                    o.pos = mul( UNITY_MATRIX_MVP, v.vertex );
                    o.color = v.color;
                    return o;
                }
                
                fixed4 frag (v2f i) : SV_Target { return i.color; }
                ENDCG
            }
        }
    }
    
####可视化法线
这个shader使用顶点位置和法线作为输入。法线的x、y、z元素作为颜色显示。因为法线是在[-1, 1]范围，所以调整偏移它们到[0, 1]范围：

    Shader "Debug/Normals" {
        SubShader {
            Pass {
                CGPROGRAM
                #pragma vertex vert
                #pragma fragment frag

                // vertex input: position, normal
                struct appdata {
                    float4 vertex : POSITION;
                    float3 normal : NORMAL;
                };

                struct v2f {
                    float4 pos : SV_POSITION;
                    fixed4 color : COLOR;
                };
                
                v2f vert (appdata v) {
                    v2f o;
                    o.pos = mul( UNITY_MATRIX_MVP, v.vertex );
                    o.color.xyz = v.normal * 0.5 + 0.5;
                    o.color.w = 1.0;
                    return o;
                }
                
                fixed4 frag (v2f i) : SV_Target { return i.color; }
                ENDCG
            }
        }
    }

####可视化切线(tangents)和次法线(binormals)
切线和次法线用于法线映射。Unity中只有切线向量存储在顶点，而次法线是通过法线和切线推出的。

这个shader使用顶点位置和切线作为输入。切线的x、y、z元素作为颜色显示。

    Shader "Debug/Tangents" {
        SubShader {
            Pass {
                CGPROGRAM
                #pragma vertex vert
                #pragma fragment frag

                // vertex input: position, tangent
                struct appdata {
                    float4 vertex : POSITION;
                    float4 tangent : TANGENT;
                };

                struct v2f {
                    float4 pos : SV_POSITION;
                    fixed4 color : COLOR;
                };
                
                v2f vert (appdata v) {
                    v2f o;
                    o.pos = mul( UNITY_MATRIX_MVP, v.vertex );
                    o.color = v.tangent * 0.5 + 0.5;
                    return o;
                }
                
                fixed4 frag (v2f i) : SV_Target { return i.color; }
                ENDCG
            }
        }
    }
    
下面的shader显示次法线。使用顶点位置、法线和切线作为输入。

    Shader "Debug/Bitangents" {
        SubShader {
            Pass {
                Fog { Mode Off }
                CGPROGRAM
                #pragma vertex vert
                #pragma fragment frag

                // vertex input: position, normal, tangent
                struct appdata {
                    float4 vertex : POSITION;
                    float3 normal : NORMAL;
                    float4 tangent : TANGENT;
                };

                struct v2f {
                    float4 pos : SV_POSITION;
                    float4 color : COLOR;
                };
                
                v2f vert (appdata v) {
                    v2f o;
                    o.pos = mul( UNITY_MATRIX_MVP, v.vertex );
                    // calculate bitangent
                    float3 bitangent = cross( v.normal, v.tangent.xyz ) * v.tangent.w;
                    o.color.xyz = bitangent * 0.5 + 0.5;
                    o.color.w = 1.0;
                    return o;
                }
                
                fixed4 frag (v2f i) : SV_Target { return i.color; }
                ENDCG
            }
        }
    }
    
    
    
---			
> 参考：<br>
[Providing vertex data to vertex programs](http://docs.unity3d.com/Manual/SL-VertexProgramInputs.html)