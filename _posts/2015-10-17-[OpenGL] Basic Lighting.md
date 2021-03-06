---
layout: post
title: OpenGL 光照基础
description: "OpenGL光照基础理论"
modified: 2015-10-17
tags: [OpenGL]
---

## Phong光照模型

现实世界的光照是非常复杂的，我们目前是没有能力完全模拟的。因此OpenGL的光照仅使用了简化的模型并基于对现实的估计来进行模拟。这些光照模型都是基于我们对光的物理特性的理解。
其中一个模型被称为冯氏光照模型(Phong Lighting Model)。冯氏光照模型的主要结构由3个元素组成：环境(Ambient)、漫反射(Diffuse)和镜面(Specular)光照。这些光照元素看起来像下面这样：

![]({{ site.url }}/images/post/2015-10-17/OpenGL Lighting 1.png)

- **环境光照(Ambient)**：即使在黑暗的情况下，世界上也仍然有一些光亮(月亮、一个来自远处的光)，所以物体永远不会是完全黑暗的。我们使用环境光照来模拟这种情况，也就是无论如何永远都给物体一些颜色。
- **漫反射光照(Diffuse)**：模拟一个发光物对物体的方向性影响(Directional Impact)。它是冯氏光照模型最显著的组成部分。面向光源的一面比其他面会更亮。
- **镜面光照(Specular)**：模拟有光泽物体上面出现的亮点。镜面光照的颜色，相比于物体的颜色更倾向于光的颜色。

## 环境光照(Ambient Lighting)
通常我们周围有很多光源，即使它们有些不是那么明显。光有一个特性是，可以向很多方向发散和反射(Reflect)到其他表面，一个物体的光照可能受到一个非直射光的影响。考虑到这种情况的算法叫做**全局照明(Global Illumination)**算法，但是这种算法既开销高昂又极其复杂。

我们会使用一种简化的全局照明模型，叫做环境光照。我们使用一个很小的常量光颜色添加进物体片段的最终颜色里，使其就算没有直射光也始终存在着一些发散的光。

    void main()
    {
        float ambientStrength = 0.1f;
        vec3 ambient = ambientStrength * lightColor;
        vec3 result = ambient * objectColor;
        color = vec4(result, 1.0f);
    }


## 漫反射光照(Diffuse Lighting)
环境光本身不提供最明显的光照效果，但是漫反射光照会对物体产生显著的视觉影响。漫反射光使物体上与光线排布越近的片段越能从光源处获得更多的亮度。为了更好的理解漫反射光照，请看下图：

![]({{ site.url }}/images/post/2015-10-17/OpenGL Lighting 2.png)

左上方有一个光源，它所发出的光线落在物体的一个片段上。我们需要测量这个光线与它所接触片段之间的角度。如果光线垂直于物体表面，这束光对物体的影响会最大化(更亮)。为了测量光线和片段的角度，我们使用法向量(Normal Vector)。

我们知道两个单位向量的角度越小，它们点乘的结果越倾向于1。当两个向量的角度是90度的时候，点乘会变为0。这同样适用于θ，θ越大，光对片段颜色的影响越小。

点乘返回一个标量，我们可以用它计算光线对片段颜色的影响，基于不同片段所朝向光源的方向的不同，这些片段被照亮的情况也不同。

- 计算法向量(Normal Vector)
  法向量是垂直于顶点表面的向量。由于顶点自身并没有表面，我们利用顶点周围的顶点计算出这个顶点的表面。可以使用叉乘来计算所有的顶点法线，但是由于3D立方体不是一个复杂的形状，所以我们可以简单的把法线数据手动添加到顶点数据中。
	
- 计算光线向量
  光线向量只需要使用光源位置与顶点位置相减即可。光源位置是固定的。顶点位置在顶点着色器中将顶点坐标与模型坐标相乘转换到世界坐标，再传递到fragment shader使用即可。
	
- 计算漫反射光
  已经计算出物体法向量norm，光线向量lightDir，则

    float diff = max(dot(norm, lightDir), 0.0);
    vec3 diffuse = diff * lightColor;

  先计算散射因子diff，因为若两个向量夹角大于90度，点乘会变成负数，负的颜色是没有实际定义的，所以为了避免，使用max来确保大于0。
	
  最终颜色为

    vec3 result = (ambient + diffuse) * objectColor;
    color = vec4(result, 1.0f);

	
- 正规矩阵：
  我们都是在世界空间坐标中进行计算的，所以，法向量也要转换为世界空间坐标，但是这不是简单地把它乘以一个模型矩阵就能搞定的。
	
  首先，法向量只是一个方向向量，不能表达空间中的特定位置。同时，法向量没有齐次坐标(w分量)。这意味着，平移不应该影响到法向量。因此，如果我们打算把法向量乘以一个模型矩阵，我们就要把模型矩阵左上角的3×3矩阵从模型矩阵中移除(设为0)，它是模型矩阵的平移部分。也可以把法向量的w分量设置为0，再乘以4×4矩阵，同样可以移除平移。对于法向量，我们只能对它应用缩放(Scale)和旋转(Rotation)变换。
	
  其次，如果模型矩阵执行了不等比缩放，法向量就不再垂直于表面了。因此，我们不能用这样的模型矩阵去乘以法向量。下面的图展示了应用了不等比缩放的矩阵对法向量的影响：	
  
  ![]({{ site.url }}/images/post/2015-10-17/OpenGL Lighting 3.png)
	
  当我们提交一个不等比缩放，法向量就不会再垂直于它们的表面了，这样光照会被扭曲。
  (注意：等比缩放不会破坏法线，因为法线的方向没被改变，而法线的长度很容易通过标准化进行修复)。
	
  修复这个行为的诀窍是使用另一个为法向量专门定制的模型矩阵。这个矩阵称之为**正规矩阵(Normal Matrix)**。
	
  正规矩阵被定义为“模型矩阵左上角的逆矩阵的转置矩阵”。注意，定义正规矩阵的大多资源就像应用到模型观察矩阵(Model-view Matrix)上的操作一样，但是由于我们只在世界空间工作(而不是在观察空间)，我们只使用模型矩阵。
	
  在顶点着色器中，我们可以使用inverse和transpose函数自己生成正规矩阵，inverse和transpose函数对所有类型矩阵都有效。注意，我们也要把这个被处理过的矩阵强制转换为3×3矩阵，这是为了保证它失去了平移属性，之后它才能乘以法向量。

    Normal = mat3(transpose(inverse(model))) * normal;


  注意：
  对于着色器来说，逆矩阵是一种开销比较大的操作，因此，在着色器应该尽量避免逆操作，因为它们必须为你场景中的每个顶点进行这样的处理。在绘制之前，最好用CPU计算出正规矩阵，然后通过uniform把值传递给着色器(和模型矩阵一样)。
	
## 镜面光照(Specular Lighting)
和环境光照一样，镜面光照同样依据光的方向向量和物体的法向量，但是不同的是它会依据观察方向。镜面光照根据光的反射特性。如果我们想象物体表面像一面镜子一样，那么，无论我们从哪里去看那个表面所反射的光，镜面光照都会达到最大化。你可以从下面的图片看到效果：

![]({{ site.url }}/images/post/2015-10-17/OpenGL Lighting 4.png)

我们计算反射向量和视线方向的角度，如果之间的角度越小，那么镜面光的作用就会越大。它的作用效果就是，当我们去看光被物体所反射的那个方向的时候，我们会看到一个高光。

观察向量可以使用观察者世界空间位置(Viewer's World Space Position)和片段的位置来计算。之后，我们计算镜面光亮度，用它乘以光的颜色，在用它加上作为之前计算的光照颜色。
也可以在观察空间(View Space)进行光照计算，这样观察者的坐标永远是(0, 0, 0)。

先定义一个高光强度：

    float specularStrength = 0.5f;

然后计算视线方向和沿法线轴的反射向量：

    vec3 viewDir = normalize(viewPos - FragPos);
    vec3 reflectDir = reflect(-lightDir, norm);

reflect函数要求的第一个是从光源指向片段位置的向量，第二个参数要求是一个法向量。

计算镜面亮度：

    float spec = pow(max(dot(viewDir, reflectDir), 0.0), 32);
    vec3 specular = specularStrength * spec * lightColor;

我们先计算视线方向与反射方向的点乘(确保它不是负值)，然后得到它的32次幂。这个32是高光的发光值(Shininess)。一个物体的发光值越高，反射光的能力越强，散射得越少，高光点越小。在下面的图片里，你会看到不同发光值对视觉(效果)的影响：

![]({{ site.url }}/images/post/2015-10-17/OpenGL Lighting 5.png)

最后一件事情是把它添加到环境光颜色和散射光颜色里，然后再乘以物体颜色：

    vec3 result = (ambient + diffuse + specular) * objectColor;
    color = vec4(result, 1.0f);
    

## Gouraud着色
早期的光照着色器，开发者在顶点着色器中实现冯氏光照。这样的优势是，相比片段来说，顶点要少得多，因此会更高效。然而，顶点着色器中的颜色值是只是顶点的颜色值，片段的颜色值是它与周围的颜色值的插值。结果就是这种光照看起来不会非常真实，除非使用了大量顶点。
在顶点着色器中实现的冯氏光照模型叫做Gouraud着色，而不是冯氏着色。由于插值，这种光照连起来有点逊色。冯氏着色能产生更平滑的光照效果。


---
> 参考：
[LearnOpenGL-CN 光照基础](http://learnopengl-cn.readthedocs.org/zh/latest/02%20Lighting/02%20Basic%20Lighting/)