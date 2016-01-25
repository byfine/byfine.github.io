---
layout: post
title: Shader：Surface Shaders
description: "Surface Shaders简介"
modified: 2015-11-22
tags: [Shader]
---

Unity中的Surface Shaders可以说是一种代码生成手段，比起顶点/像素着色器，可以让我们更轻松的编写光照shader。它并没有特殊的地方，只是生成一些重复性的代码，你仍需要使用Cg / HLSL 编写代码。

###工作原理
定义一个“surface 函数”，它会接收UVs或你需要的数据作为输入，并填充输出结构体SurfaceOutput。SurfaceOutput基本描述了表面的属性（albedo color, normal, emission, specularity等）。
然后表面着色器的编译器会算出，需要哪些输入，填充哪些输出等等，并生成实际的顶点/像素着色器代码，并渲染通道来控制正向或延迟渲染。

标准的SurfaceOutput结构体如下：

    struct SurfaceOutput
    {
        fixed3 Albedo;  // diffuse color
        fixed3 Normal;  // tangent space normal, if written
        fixed3 Emission;
        half Specular;  // specular power in 0..1 range
        fixed Gloss;    // specular intensity
        fixed Alpha;    // alpha for transparencies
    };

Unity5中， 也可以使用基于物理的光照模型。内建的 Standard 和 StandardSpecular 光照模型使用如下输出结构体：

    struct SurfaceOutputStandard
    {
        fixed3 Albedo;      // base (diffuse or specular) color
        fixed3 Normal;      // tangent space normal, if written
        half3 Emission;
        half Metallic;      // 0=non-metal, 1=metal
        half Smoothness;    // 0=rough, 1=smooth
        half Occlusion;     // occlusion (default 1)
        fixed Alpha;        // alpha for transparencies
    };
    struct SurfaceOutputStandardSpecular
    {
        fixed3 Albedo;      // diffuse color
        fixed3 Specular;    // specular color
        fixed3 Normal;      // tangent space normal, if written
        half3 Emission;
        half Smoothness;    // 0=rough, 1=smooth
        half Occlusion;     // occlusion (default 1)
        fixed Alpha;        // alpha for transparencies
    };


###Surface Shader编译指令

和其他shader一样，Surface shaders置于 CGPROGRAM .. ENDCG 区域中。区别在于：

1. 它必须被置于SubShader中，而不是Pass中。表面着色器会自动编译为多个pass。
2. 它使用#pragma surface ... 指令来表明它是Surface Shader。

\#pragma surface指令如下：

    #pragma surface surfaceFunction lightModel [optionalparams]

###必要参数:   
  
####surfaceFunction - 包含表面着色器代码的cg函数。     
函数声明如：void surf (Input IN, inout SurfaceOutput o)
Input是你定义的结构体，应该包含纹理坐标和其他需要的自动变量。
	
####lightModel - 使用的光照模型。     
内建的光照模型是基于物理的 Standard 和 StandardSpecular，也有简单的非物理的 Lambert (diffuse) 和 BlinnPhong (specular)。也可以编写自己的光照模型。 
    
- Standard 使用 SurfaceOutputStandard 输出结构体, 并匹配 Standard (metallic workflow) shader.
- StandardSpecular 使用 SurfaceOutputStandardSpecular  输出结构体, 并匹配 Standard (specular setup) shader.
- Lambert 和 BlinnPhong 是不基于物理的(通常用于Unity 4.x), 但在低端硬件上使用它们会更高效。

###可选参数    

####Transparency and alpha testing    
这是由 alpha 和 alphatest 指令控制的。透明（Transparency ）通常有两种方式：传统的alpha blending 或者 物理模拟的 premultiplied blending (允许半透明表面拥有合适的高光反射)。激活semitransparency 会使生成的代码包含blending命令，而激活 alpha cutout 在生成的片段着色器中会根据给定变量进行片段抛弃。

- alpha or alpha:auto - Will pick fade-transparency (same as alpha:fade) for simple lighting functions, and premultiplied transparency (same as alpha:premul) for physically based lighting functions.
- alpha:fade - Enable traditional fade-transparency.
- alpha:premul - Enable premultiplied alpha transparency.
- alphatest:VariableName - Enable alpha cutout transparency. Cutoff value is in a float variable with VariableName. You’ll likely also want to use addshadow directive to generate proper shadow caster pass.
- keepalpha - By default opaque surface shaders write 1.0 (white) into alpha channel, no matter what’s output in the Alpha of output struct or what’s returned by the lighting function. Using this option allows keeping lighting function’s alpha value even for opaque surface shaders.
- decal:add - Additive decal shader (e.g. terrain AddPass). This is meant for objects that lie atop of other surfaces, and use additive blending. See Surface Shader Examples
- decal:blend - Semitransparent decal shader. This is meant for objects that lie atop of other surfaces, and use alpha blending. See Surface Shader Examples。
	
####Custom modifier functions          
可以改变或计算输入的顶点数据，或改变最终计算出的片段颜色(fragment color)。    

- vertex:VertexFunction - Custom vertex modification function. This function is invoked at start of generated vertex shader, and can modify or compute per-vertex data. See Surface Shader Examples.
- finalcolor:ColorFunction - Custom final color modification function. See Surface Shader Examples.
- finalgbuffer:ColorFunction - Custom deferred path for altering gbuffer content.
- finalprepass:ColorFunction - Custom prepass base path.

####Shadows and Tessellation      
额外的指令，可用于控制shadows 和 tessellation（曲面细分）。
    
- addshadow - Generate a shadow caster pass. Commonly used with custom vertex modification, so that shadow casting also gets any procedural vertex animation. Often shaders don’t need any special shadows handling, as they can just use shadow caster pass from their fallback.
- fullforwardshadows - Support all light shadow types in Forward rendering path. By default shaders only support shadows from one directional light in forward rendering (to save on internal shader variant count). If you need point or spot light shadows in forward rendering, use this directive.
- tessellate:TessFunction - use DX11 GPU tessellation; the function computes tessellation factors. See Surface Shader Tessellationfor details.
    
####Code generation options       
默认生成的表面着色器代码会尝试处理所有可能的lighting / shadowing /lightmap 情景。然而有时有些并不需要，这时可以调整生成代码跳过它们。 

- exclude_path:deferred, exclude_path:forward, exclude_path:prepass - Do not generate passes for given rendering path (Deferred Shading, Forward and Legacy Deferred respectively).
- noshadow - Disables all shadow receiving support in this shader.
- noambient - Do not apply any ambient lighting or light probes.
- novertexlights - Do not apply any light probes or per-vertex lights in Forward rendering.
- nolightmap - Disables all lightmapping support in this shader.
- nodynlightmap - Disables runtime dynamic global illumination support in this shader.
- nodirlightmap - Disables directional lightmaps support in this shader.
- nofog - Disables all built-in Fog support.
- nometa - Does not generate a “meta” pass (that’s used by lightmapping & dynamic global illumination to extract surface information).
- noforwardadd - Disables Forward rendering additive pass. This makes the shader support one full directional light, with all other lights computed per-vertex/SH. Makes shaders smaller as well.

####Miscellaneous options         
杂项设置

- softvegetation - Makes the surface shader only be rendered when Soft Vegetation is on.
- interpolateview - Compute view direction in the vertex shader and interpolate it; instead of computing it in the pixel shader. This can make the pixel shader faster, but uses up one more texture interpolator.
- halfasview - Pass half-direction vector into the lighting function instead of view-direction. Half-direction will be computed and normalized per vertex. This is faster, but not entirely correct.
- approxview - Removed in Unity 5.0. Use interpolateview instead.
- dualforward - Use dual lightmaps in forward rendering path.
	

###表面着色器的输入结构体：
输入的结构体 Input 通常包含所需的纹理坐标。纹理坐标通常用uv_加贴图名字构成。（或者uv2_ 表示第二张贴图）。
还有其他值可以放入Input结构体：

- float3 viewDir - will contain view direction, for computing Parallax effects, rim lighting etc.   
观察向量。
- float4 with COLOR semantic - will contain interpolated per-vertex color.  
每个顶点的插值颜色。
- float4 screenPos - will contain screen space position for reflection or screenspace effects.  
屏幕空间坐标。
- float3 worldPos - will contain world space position.  
世界空间坐标。
- float3 worldRefl - will contain world reflection vector if surface shader does not write to o.Normal. See Reflect-Diffuse shader for example.     
世界中的反射向量。
- float3 worldNormal - will contain world normal vector if surface shader does not write to o.Normal.       
世界中的法线向量。
- float3 worldRefl; INTERNAL_DATA - will contain world reflection vector if surface shader writes to o.Normal. To get the reflection vector based on per-pixel normal map, use WorldReflectionVector (IN, o.Normal). See Reflect-Bumped shader for example.     
内部数据。如果表面着色写入到o.Normal，将包含世界反射向量 。
- float3 worldNormal; INTERNAL_DATA - will contain world normal vector if surface shader writes to o.Normal. To get the normal vector based on per-pixel normal map, use WorldNormalVector (IN, o.Normal).      
内部数据。如果表面着色写入到o.Normal， 将包含世界法线向量。


###CG中的一些函数：

    half4 c = tex2D (_MainTex, IN.uv_MainTex); 
    
tex2D: 对一张贴图采样，返回float4

    o.Normal = UnpackNormal (tex2D (_BumpMap, IN.uv_BumpMap));
    
UnpackNormal是定义在UnityCG.cginc文件中的方法，这个文件中包含了一系列常用的CG变量以及方法。UnpackNormal接受一个fixed4的输入，并将其转换为所对应的法线值（fixed3）。在解包得到这个值之后，将其赋给输出的Normal，就可以参与到光线运算中完成接下来的渲染工作了。

    saturate(x)
    
saturate ：将x限制在0-1之间

		
		
###Surface Shaders中的自定义光照模型
当写Surface Shaders时，你会描述一个表面的属性和由光照模型计算的灯光互动。内建的光照模型是Lambert (diffuse lighting) 和 BlinnPhong (specular lighting).

但也许有时你想使用自定义的光照模型。光照模型无非就是一组符合一些规则的Cg/HLSL函数。Lambert 和 BlinnPhong 模型是定义在Lighting.cginc文件。   
(Windows：{unity install path}/Data/CGIncludes/Lighting.cginc, Mac：/Applications/Unity/Unity.app/Contents/CGIncludes/Lighting.cginc).

####光照模型声明  
光照模型是一组以Lighting开头的有规则的函数，它们可以声明在shader的任何位置或任一包含文件内。这些函数有：
     
    //用于光照模型的forward rendering path，不依赖视线方向（如diffuse）。     
    half4 Lighting<Name> (SurfaceOutput s, half3 lightDir, half atten);
          
    //用于光照模型的forward rendering path，但是赖视线方向。
    half4 Lighting<Name> (SurfaceOutput s, half3 lightDir, half3 viewDir, half atten);
           
    //用于deferred lighting path。    
    half4 Lighting<Name>_PrePass (SurfaceOutput s, half4 light);
         	
注意，你并不需要声明所有函数。一个光照模型既可以使用视角方向也可以不使用。同样，如果光照模型不用于deferred lighting，你并不需要 _PrePass 函数。
	
####解码光照贴图
与灯光函数类似，解码光照贴图，可以根据是否依赖视角方向选择使用下面一种函数。要解码标准的unity光照贴图数据（传入color, totalColor, indirectOnlyColor和scale参数），使用DecodeLightmap函数。

自定义的光照贴图解码函数会自己处理forward 和 deferred lighting rendering paths。但你必须了解，对于deferred，Lighting<Name>_PrePass 函数必须在光照贴图解码之后调用，然后light参数会包含实时灯光和光照贴图的和。如果必要的话，可以使用内建宏UNITY_PASS_PREPASSFINAL来区分foward和deferrd。

解码 Single lightmaps 的函数：    

    //用于不依赖观察方向的光照模型。
    half4 Lighting<Name>_SingleLightmap (SurfaceOutput s, fixed4 color);         
        
    //依赖观察方向的光照模型。
    half4 Lighting<Name>_SingleLightmap (SurfaceOutput s, fixed4 color, half3 viewDir); 

	
解码 Dual lightmaps 的函数：      
    
    //用于不依赖观察方向的光照模型。
    half4 Lighting<Name>_DualLightmap (SurfaceOutput s, fixed4 totalColor, fixed4 indirectOnlyColor, half indirectFade);     
    
    //依赖观察方向的光照模型。
    half4 Lighting<Name>_DualLightmap (SurfaceOutput s, fixed4 totalColor, fixed4 indirectOnlyColor, half indirectFade, half3 viewDir);     
     

解码 Directional lightmaps 的函数：               

    //用于不依赖观察方向的光照模型。
    half4 Lighting<Name>_DirLightmap (SurfaceOutput s, fixed4 color, fixed4 scale, bool surfFuncWritesNormal);    
       
    //依赖观察方向的光照模型。
    half4 Lighting<Name>_DirLightmap (SurfaceOutput s, fixed4 color, fixed4 scale, half3 viewDir, bool surfFuncWritesNormal, out half3 specColor); 
          

---			
> 参考：<br>
[Writing Surface Shaders](http://docs.unity3d.com/Manual/SL-SurfaceShaders.html)<br>
[Custom Lighting models in Surface Shaders](http://docs.unity3d.com/Manual/SL-SurfaceShaderLighting.html)
