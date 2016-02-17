---
layout: post
title: Shader：Ramp Texture 的应用
description: "Ramp Texture 的应用"
modified: 2015-11-23
tags: [Shader]
---

Ramp Texture 是一种控制漫反射的方法。
一般Ramp Texture是一种类似下图的渐变图：

![]({{ site.url }}/images/post/Shader/Ramp-Texutre-1.png)

### 一般的漫反射光照模型

这是一个半兰伯特光照模型：

    half4 LightingWrapLambert (SurfaceOutput s, half3 lightDir, half atten) {
        // 计算光线与法线的夹角余弦，求漫反射颜色。夹角越小，值越大。
        half NdotL = dot (s.Normal, lightDir);

        // 将结果都转换到[0,1]范围，可以保证光线和平面夹角大于90度（即余弦值小于0）时也可以有颜色。
        half diff = NdotL * 0.5 + 0.5;

        half4 c;
        // atten是光照的衰减值
        c.rgb = s.Albedo * _LightColor0.rgb * (diff * atten);
        c.a = s.Alpha;
        return c;
    }

加入Ramp Texture：

    half4 LightingRamp (SurfaceOutput s, half3 lightDir, half atten) {
        …
        …
        // 根据一张ramp贴图，根据余弦值对其颜色采样
        half3 ramp = tex2D (_RampTex, float2(diff, diff)).rgb;
        
        half4 c;
        // atten是光照的衰减值
        c.rgb = s.Albedo * _LightColor0.rgb * ramp * atten;
        c.a = s.Alpha;
        return c;
    }
        
注意  half3 ramp = tex2D (_RampTex, float2(diff, diff)).rgb;      
我们使用了diff值对RampTex 进行采样。        
因为现在使用的Ramp贴图可以视为1D的颜色索引值，所以float2(diff, diff)也可以是float2(diff, 0)，仅对x方向采样。      
我们知道diff表示了光线和法线的夹角，夹角越小，diff值越大，颜色越亮，光线越直射表面。      
所以效果上看，就是光线直接照射的面，采样到越靠右的颜色。


![]({{ site.url }}/images/post/Shader/Ramp-Texutre-Demo-1.png)

![]({{ site.url }}/images/post/Shader/Ramp-Texutre-Demo-2.jpg)

使用一个spotlight照射方块，Ramp贴图如上，可以看到光直接照射的面，diff很大，采样最右边的颜色，侧面的diff很小，采样倒了左侧的颜色。
