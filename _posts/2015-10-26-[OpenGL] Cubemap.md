---
layout: post
title: OpenGL 立方体贴图(Cubemap)
description: "OpenGL 立方体贴图 简介"
modified: 2015-10-26
tags: [OpenGL]
---

## Cubemap：
之前我们一直使用的是2D纹理，下面来讨论一种将多个纹理组合起来映射到一个单一纹理的类型，它就是cubemap。

基本上说cubemap包含6个2D纹理，每个2D纹理是立方体的一个面。你可能会奇怪费事地把6个纹理结合为一个这样的立方体有什么用？这是因为cubemap有自己特有的属性，可以使用一个方向向量对它们索引和采样。
想象一下，我们有一个1×1×1的单位立方体，在它内部有个以原点为起点的方向向量。

从cubemap上使用橘黄色向量采样一个纹理值看起来和下图有点像：

![]({{ site.url }}/images/post/2015-10-26/OpenGL Cubemap 1.png)

方向向量的大小不重要。一旦提供了方向，OpenGL就会获取方向向量触碰到立方体表面上的相应的纹理像素（texel），这样就返回了正确的纹理采样值。

方向向量触碰到立方体表面的一点也就是cubemap的纹理位置，这意味着只要立方体的中心位于原点上，我们就可以使用立方体的位置向量来对cubemap进行采样。然后我们就可以获取所有顶点的纹理坐标，就和立方体上的顶点位置一样。所获得的结果是一个纹理坐标，通过这个纹理坐标就能获取到cubemap上正确的纹理。


## 创建一个Cubemap：
Cubemap和其他纹理一样，所以要创建一个cubemap，在进行任何纹理操作之前，需要生成一个纹理，激活相应纹理单元然后绑定到合适的纹理目标上。
这次要绑定到 GL_TEXTURE_CUBE_MAP纹理类型：

    GLuint textureID;
    glGenTextures(1, &textureID);
    glBindTexture(GL_TEXTURE_CUBE_MAP, textureID);

由于cubemap包含6个纹理，我们必须调用glTexImage2D函数6次，还需要把纹理目标（target）参数设置为cubemap特定的面，来告诉OpenGL创建的纹理是对应立方体哪个面的。
OpenGL就提供了6个不同的纹理目标，来应对cubemap的各个面：

![]({{ site.url }}/images/post/2015-10-26/OpenGL Cubemap 2.png)

根据枚举特效，可以每次加1进行循环操作

    for(GLuint i = 0; i < textures_paths.size(); i++)
    {
        image = SOIL_load_image(textures_paths[i], &width, &height, 0, SOIL_LOAD_RGB);
        glTexImage2D(GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, 0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, image);
    }


我们也要定义它的环绕方式和过滤方式：

    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_EDGE);


在片段着色器中，我们使用一个不同的采样器 — samplerCube，用它来从texture函数中采样，但是这次使用的是一个vec3方向向量，取代vec2。下面是一个片段着色器使用了cubemap的例子：

    in vec3 textureDir; // 用一个三维方向向量来表示Cubemap纹理的坐标
    uniform samplerCube cubemap;  // Cubemap纹理采样器
    void main() {
        color = texture(cubemap, textureDir);
    }


## 天空盒(Skybox)：
cubemap可以简单的实现很多有意思的技术。其中之一便是天空盒(Skybox)。天空盒是一个包裹整个场景的立方体，它由6个图像构成一个环绕的环境，给玩家一种他所在的场景比实际的要大得多的幻觉。如果把这6个面折叠到一个立方体中，就会获得模拟了一个巨大风景的立方体。

加载天空盒：
先按上面方法加载6张纹理。然后创建一个立方体来绘制天空盒。当一个立方体的中心位于原点(0，0，0)的时候，它的每一个位置向量也就是以原点为起点的方向向量。这个方向向量就是我们要得到的立方体某个位置的相应纹理值。所以我们只需要提供位置向量，而无需纹理坐标。它的顶点着色器如下：

    #version 330 core
    layout (location = 0) in vec3 position;
    out vec3 TexCoords;

    uniform mat4 projection;
    uniform mat4 view;

    void main()
    {
        gl_Position =   projection * view * vec4(position, 1.0);  
        TexCoords = position;
    }

顶点着色器把输入的位置向量作为输出给片段着色器的纹理坐标。片段着色器就会把它们作为输入去采样samplerCube：

    #version 330 core
    in vec3 TexCoords;
    out vec4 color;

    uniform samplerCube skybox;

    void main()
    {
        color = texture(skybox, TexCoords);
    }

我们希望天空盒以玩家为中心。移除视图矩阵的平移部分，这样移动就影响不到天空盒的位置向量了。我们可以只用4X4矩阵的3×3部分去除平移。简单地将矩阵转为33矩阵再转回来，就能达到目标：

    glm::mat4 view = glm::mat4(glm::mat3(camera.GetViewMatrix()));


为了绘制天空盒，我们可以把它作为场景中第一个绘制的物体并且关闭深度写入。这样天空盒才能成为所有其他物体的背景来绘制出来。

但这样并不高效，如果先渲染天空盒，那么就会为屏幕上每一个像素运行片段着色器，即使前面有物体遮挡。但如果我们直接打开深度测试，因为天空盒是个1×1×1的立方体，极有可能会通不过深度测试。办法是，我们需要让深度缓冲相信天空盒的深度缓冲有着最大深度值1.0，如此只要有个物体存在深度测试就会失败，看似物体就在它前面了。

前面坐标系统我们说过，透视除法（perspective division）是在顶点着色器运行之后执行的，它把gl_Position的xyz坐标除以w元素。而除法结果的z就等于顶点的深度值。因此，我们可以把输出位置的z元素设置为它的w元素，这样它的z就会转换为w/w = 1.0。修改顶点着色器：

    void main()
    {
        vec4 pos = projection * view * vec4(position, 1.0);
        gl_Position = pos.xyww;
        TexCoords = position;
    }

1.0就是深度值的最大值，只有在没有任何物体可见的情况下天空盒才会被渲染（只有通过深度测试才渲染，否则假如有任何物体存在，就不会被渲染，只去渲染物体）。

我们必须改变一下深度方程，把它设置为GL_LEQUAL，原来默认的是GL_LESS。深度缓冲会为天空盒用1.0这个值填充深度缓冲，所以我们需要保证天空盒是使用小于等于深度缓冲来通过深度测试的，而不是小于。


---
> 参考
[LearnOpenGL-CN 立方体贴图](http://learnopengl-cn.readthedocs.org/zh/latest/04%20Advanced%20OpenGL/06%20Cubemaps/)