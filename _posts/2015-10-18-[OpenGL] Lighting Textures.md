---
layout: post
title: OpenGL 光照贴图
description: "光照贴图"
modified: 2015-10-18
tags: [OpenGL]
---

前面我们为一个物体定义了一个材质，但是现实中物体通常不会只有这么一种材质，一个物体每个部分都可能有多种材质属性。
前面的材质系统除了对最简单的模型外都是不够的，所以我们需要扩展前面的系统，我们要介绍diffuse和specular贴图。它们允许你对一个物体的diffuse（而对于简洁的ambient成分来说，它们几乎总是是一样的）和specular成分能够有更精确的影响。

## 漫反射贴图
我们希望通过某种方式对每个原始像素独立设置diffuse颜色。这与纹理的道理是一样的。
使用一张图片包裹住物体，我们为每个原始像素索引独立颜色值。在有光的场景里，通常叫做**漫反射贴图**(Diffuse texture)，因为这个纹理图像表现了所有物体的diffuse颜色。

我们把纹理储存为sampler2D，并在Material结构体中。我们使用diffuse贴图替代之前定义的vec3类型的diffuse颜色。

下面是一张带钢圈的木箱贴图：

![]({{ site.url }}/images/post/2015-10-18/OpenGL LightTex 1.png)


我们也要移除amibient材质颜色向量，因为ambient颜色绝大多数情况等于diffuse颜色，所以不需要分别去储存它：

    struct Material
    {
        sampler2D diffuse;
        vec3 specular;
        float shininess;
    };
    ...
    in vec2 TexCoords;

然后我们简单地从纹理采样，来获得原始像素的diffuse颜色值：

    vec3 diffuse = light.diffuse * diff * vec3(texture(material.diffuse, TexCoords));

同样，不要忘记把ambient材质的颜色设置为diffuse材质的颜色：

    vec3 ambient = light.ambient * vec3(texture(material.diffuse, TexCoords));


## 镜面贴图
如果我们的物体是个木箱子，我们知道木头是不应该有镜面高光的。通过把物体设置specular材质设置为vec3(0.0f)来修正它。但是如果木箱贴图带铁边，这将会不再显示镜面高光，我们知道钢铁是会显示一些镜面高光的。我们会想要控制物体部分地显示镜面高光，它带有修改了的亮度。

我们同样用一个纹理贴图，来获得镜面高光。这意味着我们需要生成一个黑白（或者你喜欢的颜色）纹理来定义specular亮度，把它应用到物体的每个部分。下面是一个specular贴图的例子：

![]({{ site.url }}/images/post/2015-10-18/OpenGL LightTex 2.png)

一个specular高光的亮度可以通过图片中每个纹理的亮度来获得。specular贴图的每个像素可以显示为一个颜色向量，比如：黑色代表颜色向量vec3(0.0f)，灰色是vec3(0.5f)。在片段着色器中，我们采样相应的颜色值，把它乘以光的specular亮度。像素越“白”，乘积的结果越大，物体的specualr部分越亮。

由于箱子几乎是由木头组成，木头作为一个材质不会有镜面高光，整个不透部分的diffuse纹理被用黑色覆盖：黑色部分不会包含任何specular高光。箱子的铁边有一个修改的specular亮度，它自身更容易受到镜面高光影响，木纹部分则不会。

使用Photoshop之类的工具，剪切一些部分，非常容易变换一个diffuse纹理为specular图片，以增加亮度/对比度的方式，可以把这个部分变换为黑色或白色。

同样要更新片断着色器中的材质：


    struct Material
    {
        sampler2D diffuse;
        sampler2D specular;
        float shininess;
    };


计算specular：

    vec3 specular = light.specular * spec * vec3(texture(material.specular, TexCoords));

你也可以在specular贴图里使用颜色，不单单为每个原始像素设置specular亮度，同时也设置specular高光的颜色。从真实角度来说，specular的颜色基本是由光源自身决定的，所以它不会生成真实的图像（这就是为什么图片通常是黑色和白色的：我们只关心亮度）。



---
> 参考
[LearnOpenGL-CN 光照贴图](http://learnopengl-cn.readthedocs.org/zh/latest/02%20Lighting/04%20Lighting%20maps/)