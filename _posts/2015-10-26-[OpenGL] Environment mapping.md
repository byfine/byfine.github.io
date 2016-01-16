---
layout: post
title: OpenGL 环境映射（Environment mapping）
description: "OpenGL 环境映射 简介"
modified: 2015-10-26
tags: [OpenGL]
---

利用cubemap，我们拥有了把整个环境映射为一个单独纹理的对象，我们利用这个信息能做的不仅是天空盒。使用带有场景环境的cubemap，我们还可以让物体有一个反射或折射属性。像这样使用了环境cubemap的技术叫做**环境映射**（environment mapping ）技术，其中最重要的两个是反射(reflection)和折射(refraction)。

##反射(reflection)：
反射就是一个物体（或物体的某部分）反射他周围环境的属性，比如物体的颜色多少有些等于它周围的环境，这要基于观察者的角度。例如一个镜子是一个反射物体：它会基于观察者的角度泛着它周围的环境。
反射的基本思路不难。下图展示了如何计算反射向量，然后使用这个向量去从一个cubemap中采样：

![反射示意]({{ site.url }}/images/post/2015-10-26/OpenGL EnvironmentMapping 1.png)

我们基于观察方向向量I和物体的法线向量N计算出反射向量R。我们可以使用GLSL的内建函数reflect来计算这个反射向量。最后向量R作为一个方向向量对cubemap进行索引/采样，返回一个环境的颜色值。最后的效果看起来就像物体反射了天空盒。

因为之前cubemap教程中，我们在场景中已经设置了一个天空盒，创建反射就不难了。我们改变一下箱子使用的那个片段着色器，给箱子一个反射属性：

    #version 330 core
    in vec3 Normal;
    in vec3 Position;
    out vec4 color;

    uniform vec3 cameraPos;
    uniform samplerCube skybox;

    void main()
    {
        vec3 I = normalize(Position - cameraPos);
        vec3 R = reflect(I, normalize(Normal));
        color = texture(skybox, R);
    }


我们先来计算观察/摄像机方向向量I，然后使用它来计算反射向量R，接着我们用R从天空盒cubemap采样。要注意的是，我们需要修正顶点着色器来提供Normal和Position。

    #version 330 core
    layout (location = 0) in vec3 position;
    layout (location = 1) in vec3 normal;

    out vec3 Normal;
    out vec3 Position;

    uniform mat4 model;
    uniform mat4 view;
    uniform mat4 projection;

    void main()
    {
        gl_Position = projection * view * model * vec4(position, 1.0f);
        Normal = mat3(transpose(inverse(model))) * normal;
        Position = vec3(model * vec4(position, 1.0f));
    }

我们使用正规矩阵(normal matrix)变换，来获得法线向量。Position输出的向量是一个世界空间位置向量，所以要乘以model矩阵。

我们可以引进反射贴图(reflection map)来使模型有另一层细节。和diffuse、specular贴图一样，我们可以从反射贴图上采样来决定fragment的反射率。使用反射贴图我们还可以决定模型的哪个部分有反射能力，以及强度是多少。

##折射(refraction)：
环境映射的另一个形式叫做折射，它和反射差不多。折射是光线通过特定材质对光线方向的改变。我们通常看到像水一样的表面，光线并不是直接通过的，而是让光线弯曲了一点。它看起来像你把半只手伸进水里的效果。

折射遵守斯涅尔定律（Snell's law），使用环境贴图看起来就像这样：

![折射示意]({{ site.url }}/images/post/2015-10-26/OpenGL EnvironmentMapping 2.png)

我们有个观察向量I，一个法线向量N，这次折射向量是R。就像你所看到的那样，观察向量的方向有轻微弯曲。弯曲的向量R随后用来从cubemap上采样。

折射可以通过GLSL的内建函数refract来实现，除此之外还需要一个法线向量，一个观察方向和一个两种材质之间的折射指数。

折射指数决定了一个材质上光线扭曲的数量，每个材质都有自己的折射指数。下表是常见的折射指数：

![]({{ site.url }}/images/post/2015-10-26/OpenGL EnvironmentMapping 3.png)

我们使用这些折射指数来计算光线通过两个材质的比率。在我们的例子中，光线/视线从空气进入玻璃的比率是1.00 / 1.52 = 0.658。

改变片段着色器：

    void main()
    {
        float ratio = 1.00 / 1.52;
        vec3 I = normalize(Position - cameraPos);
        vec3 R = refract(I, normalize(Normal), ratio);
        color = texture(skybox, R);
    }

你可以向想象一下，如果将光线、反射、折射和顶点的移动合理的结合起来就能创造出漂亮的水的图像。一定要注意，出于物理精确的考虑当光线离开物体的时候还要再次进行折射；现在我们简单的使用了单边（一次）折射，大多数目的都可以得到满足。


##动态环境贴图（Dynamic environment maps）:
现在，我们已经使用了静态图像组合的天空盒，看起来不错，但是没有考虑到物体可能移动的实际场景。我们到现在还没注意到这点，是因为我们目前还只使用了一个物体。如果我们有个镜子一样的物体，它周围有多个物体，只有天空盒在镜子中可见，和场景中只有这一个物体一样。

使用帧缓冲可以为提到的物体的所有6个不同角度创建一个场景的纹理，把它们每次渲染迭代储存为一个cubemap。之后我们可以使用这个（动态生成的）cubemap来创建真实的反射和折射表面，这样就能包含所有其他物体了。这种方法叫做动态环境映射（dynamic environment mapping）,因为我们动态地创建了一个物体的以其四周为参考的cubemap，并把它用作环境贴图。

它看起效果很好，但是有一个劣势：使用环境贴图我们必须为每个物体渲染场景6次，这需要非常大的开销。现代应用尝试尽量使用天空盒子，凡可能预编译cubemap就创建少量动态环境贴图。动态环境映射是个非常棒的技术，要想在不降低执行效率的情况下实现它就需要很多巧妙的技巧。

---
> 参考
[LearnOpenGL-CN 立方体贴图](http://learnopengl-cn.readthedocs.org/zh/latest/04%20Advanced%20OpenGL/06%20Cubemaps/)