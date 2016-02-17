---
layout: post
title: OpenGL 帧缓存（Frame buffer）
description: "OpenGL 帧缓存 简介"
modified: 2015-10-24
tags: [OpenGL]
---

## 帧缓存（Frame buffer）
帧缓存是屏幕所显示画面的一个直接映象，又称为位映射图 (Bit Map) 或光栅。帧缓存的每一存储单元对应屏幕上的一个像素，整个帧缓存对应一帧图像。

图形程序一个重要的目标，就是在屏幕上绘制图像（或者绘制到离屏的一处缓存中）。帧缓存（通常也就是屏幕）是由矩形的像素数组组成的，每个像素都可以在图像对应的点上显示一小块方形的颜色值。经过光栅化阶段，也就是执行片元着色器之后，得到的数据还不是真正的像素，只是候选的片元。每个片元都包含与像素位置对应的坐标数据，以及颜色和深度的存储值。通常来说，像素（x，y）填充的区域是以x为左侧，x+1为右侧，y为底部，而y+1为顶部的一处矩形区域。

一个支持OpenGL渲染的窗口 (即帧缓存) 可能包含以下的组合：
至多4个颜色缓存，一个深度缓存，一个模板缓存，一个积累缓存，一个多重采样缓存。

OpenGL给了我们自己定义帧缓存的自由，我们可以选择性的定义自己的颜色缓冲、深度和模板缓冲。我们目前所做的渲染操作都是是在默认的帧缓冲之上进行的。当你创建了你的窗口的时候默认帧缓冲就被创建和配置好了（GLFW为我们做了这件事）。通过创建我们自己的帧缓冲我们能够获得一种额外的渲染方式。

## 创建一个帧缓存
我们可以使用一个叫做glGenFramebuffers的函数来创建一个帧缓冲对象（简称FBO）：

    GLuint fbo;
    glGenFramebuffers(1, &fbo);

这种对象的创建和使用的方式与之前见到的差不多。先创建一个帧缓冲对象，把它绑定到当前帧缓冲，做一些操作，然后解绑帧缓冲。我们使用glBindFramebuffer来绑定帧缓冲：

    glBindFramebuffer(GL_FRAMEBUFFER, fbo);


绑定到GL_FRAMEBUFFER目标后，接下来所有的读、写帧缓冲的操作都会影响到当前绑定的帧缓冲。也可以使用GL_READ_FRAMEBUFFER或GL_DRAW_FRAMEBUFFER，把帧缓冲分开绑定到读或写目标上。

建构一个完整的帧缓冲必须满足以下条件：

- 我们必须往里面加入至少一个附件（颜色、深度、模板缓冲）。
- 其中至少有一个是颜色附件。
- 所有的附件都应该是已经完全做好的（已经存储在内存之中）。
- 每个缓冲都应该有同样数目的样本。

我们需要为帧缓冲创建一些附件，还需要把这些附件附加到帧缓冲上。然后使用下面方法检查是否完成：

    if(glCheckFramebufferStatus(GL_FRAMEBUFFER) == GL_FRAMEBUFFER_COMPLETE)

后续所有渲染操作将渲染到当前绑定的帧缓存的附加缓存中，由于我们的帧缓冲不是默认的帧缓存，渲染命令对窗口的视频输出不会产生任何影响。出于这个原因，它被称为**离屏渲染**（off-screen rendering），就是渲染到一个另外的缓存中。

如果要使渲染操作对窗口产生影响，要重新绑定0来使默认帧缓冲激活：

    glBindFramebuffer(GL_FRAMEBUFFER, 0);


当做完所有帧缓冲操作，要删除帧缓冲对象：

    glDeleteFramebuffers(1, &fbo);


在执行完成检测前，我们先把一个或更多的附件附加到帧缓冲上。一个附件就是一个内存地址，这个内存地址里面包含一个为帧缓冲准备的缓冲，它可以是个图像。当创建一个附件的时候我们有两种方式可以采用：纹理或渲染缓冲（renderbuffer）对象。

## 纹理附件（Texture attachments）
当把一个纹理附件加到帧缓冲上的时候，所有渲染命令会写入到纹理上，就像它是一个普通的颜色、深度或者模板缓冲一样。使用纹理的好处是，所有渲染操作的结果都会被储存为一个纹理图像，这样我们就可以简单的在着色器中使用了。

为帧缓存创建一个纹理和创建普通纹理差不多：

    GLuint texture;
    glGenTextures(1, &texture);
    glBindTexture(GL_TEXTURE_2D, texture);

    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, 800, 600, 0, GL_RGB, GL_UNSIGNED_BYTE, NULL);

    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);


主要的区别是我们把纹理的维度设置为屏幕大小（尽管不是必须的），我们还传递NULL作为纹理的data参数。对于这个纹理，我们只分配内存，而不去填充它。纹理填充会在渲染到帧缓冲的时候去做。同样，要注意，我们不用关心环绕方式或者Mipmap，因为在大多数时候都不会需要它们的。

如果你打算把整个屏幕渲染到一个或大或小的纹理上，你需要用新的纹理的尺寸再次调用glViewport（在渲染到你的帧缓冲前），否则只有一小部分纹理或屏幕能够绘制到纹理上。

现在我们已经创建了一个纹理，最后一件要做的事情是把它附加到帧缓冲上：

    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0,GL_TEXTURE_2D, texture, 0);


glFramebufferTexture2D 函数的参数：

- target：我们所创建的帧缓冲类型的目标（绘制、读取或两者都有）。
- attachment：我们所附加的附件的类型。现在我们附加的是一个颜色附件。需要注意，最后的那个0是暗示我们可以附加1个以上颜色的附件。
- textarget：你希望附加的纹理类型。
- texture：附加的实际纹理。
- level：Mipmap level。我们设置为0。

除颜色附件以外，我们还可以附加一个深度和一个模板纹理到帧缓冲对象上。若要附加深度缓冲类型，使用GL_DEPTH_ATTACHMENT设置附件类型。若要附加模板缓冲，要使用 GL_STENCIL_ATTACHMENT设置附加类型，同时把glTexImage2D中纹理格式指定为 GL_STENCIL_INDEX。

也可以同时附加一个深度缓冲和一个模板缓冲为一个单独的纹理。这样纹理的每32位数值就包含了24位的深度信息和8位的模板信息。可以使用GL_DEPTH_STENCIL_ATTACHMENT类型设置，下面是一个附加了深度和模板缓冲为单一纹理的例子：

    glTexImage2D( GL_TEXTURE_2D, 0, GL_DEPTH24_STENCIL8, 800, 600, 0, GL_DEPTH_STENCIL, GL_UNSIGNED_INT_24_8, NULL );
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_STENCIL_ATTACHMENT, GL_TEXTURE_2D, texture, 0);


## 渲染缓冲对象附件（Renderbuffer object attachments）
帧缓存的附件方式除了纹理，还有渲染缓冲对象（Renderbuffer objects）。和纹理一样，渲染缓冲对象也是一个缓冲，它可以是一堆字节、整数、像素或者其他东西。渲染缓冲对象的一大优点是，它以OpenGL原生渲染格式储存它的数据，因此在离屏渲染到帧缓冲的时候，这些数据就相当于被优化过的了，在写入或把它们的数据简单地到其他缓冲的时候非常快。

渲染缓冲对象将所有渲染数据直接储存到它们的缓冲里，而不会进行针对特定纹理格式的任何转换，这样它们就成了一种快速可写的存储介质了。然而，渲染缓冲对象通常是只写的，不能修改它们（就像获取纹理，不能写入纹理一样）。可以用glReadPixels函数去读取，函数返回一个当前绑定的帧缓冲的特定像素区域，而不是直接返回附件本身。

创建一个渲染缓冲对象和创建帧缓冲代码差不多：

    GLuint rbo;
    glGenRenderbuffers(1, &rbo);


相似地，把渲染缓冲对象绑定，这样所有后续渲染缓冲操作都会影响到当前的渲染缓冲对象：

    glBindRenderbuffer(GL_RENDERBUFFER, rbo);

由于渲染缓冲对象通常是只写的，它们经常作为深度和模板附件来使用。因为我们需要把深度值和模板值提供给测试，但不需要对这些值采样，所以深度和模板缓冲对象是完全符合的。当我们不去从这些缓冲中采样的时候，渲染缓冲对象通常很合适，因为它们等于是被优化过的。

调用glRenderbufferStorage函数可以创建一个深度和模板渲染缓冲对象，我们选择GL_DEPTH24_STENCIL8作为内部格式：

    glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH24_STENCIL8, 800, 600);

最后一件还要做的事情是把帧缓冲对象附加上：

    glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_STENCIL_ATTACHMENT, GL_RENDERBUFFER, rbo);


#### 比较
  在帧缓冲项目中，渲染缓冲对象可以提供一些优化，但更重要的是知道何时使用渲染缓冲对象，何时使用纹理。通常的规则是，如果你永远都不需要从特定的缓冲中进行采样，渲染缓冲对象对特定缓冲是更明智的选择。如果哪天需要从比如颜色或深度值这样的特定缓冲采样数据的话，你最好还是使用纹理附件。从执行效率角度考虑，它不会对效率有太大影响。

## 渲染到纹理
现在我们知道了一些帧缓冲工作原理，开始尝试使用它们。我们把场景渲染到一个颜色纹理上，这个纹理附加到一个我们创建的帧缓冲上，然后把这个纹理绘制到一个铺满屏幕的四边形上。输出的图像看似和没用帧缓冲一样，但其实是直接打印到了一个单独的四边形上面。为什么这很有用呢？下一部分我们会看到原因。

我们先来创建帧缓存，具体思路是：
先创建帧缓存，然后创建一个颜色纹理附件用于绘制，再创建一个深度和模板 渲染缓存对象用于深度测试（本例先不使用模板测试），还要把它们都附加到帧缓存上。

第一件要做的事情是创建一个帧缓冲对象，并绑定它，这比较明了：

    GLuint framebuffer;
    glGenFramebuffers(1, &framebuffer);
    glBindFramebuffer(GL_FRAMEBUFFER, framebuffer);

下一步我们创建一个纹理图像，这是我们将要附加到帧缓冲的颜色附件。我们把纹理的尺寸设置为窗口的宽度和高度，并保持数据未初始化：

    // Generate texture
    GLuint texColorBuffer;
    glGenTextures(1, &texColorBuffer);
    glBindTexture(GL_TEXTURE_2D, texColorBuffer);
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, 800, 600, 0, GL_RGB, GL_UNSIGNED_BYTE, NULL);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR );
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    glBindTexture(GL_TEXTURE_2D, 0);

    // 把颜色纹理附加到帧缓存上
    glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, texColorBuffer, 0);


接下来创建一个渲染缓冲对象来进行深度测试和模板测试。记住，当你不打算从指定缓冲采样的的时候，渲染缓冲对象是不错的选择。我们把它设置为GL_DEPTH24_STENCIL8，对于我们的目的来说这个精确度已经足够了。

    GLuint rbo;
    glGenRenderbuffers(1, &rbo);
    glBindRenderbuffer(GL_RENDERBUFFER, rbo);
    glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH24_STENCIL8, 800, 600);  
    glBindRenderbuffer(GL_RENDERBUFFER, 0);

    // 把渲染缓冲对象附加到 帧缓冲
    glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_STENCIL_ATTACHMENT, GL_RENDERBUFFER, rbo);


然后我们检查帧缓冲是否完成了：

    if(glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE){
        cout << "ERROR::FRAMEBUFFER:: Framebuffer is not complete!" << endl;
    }
    // 解绑帧缓冲，确保不会意外渲染到错误的帧缓冲上。
    glBindFramebuffer(GL_FRAMEBUFFER, 0);


现在完成帧缓存了，要做的就是渲染到帧缓存上。具体流程如下：

1. 绑定我们创建的帧缓存来激活它，开启深度测试。
2. 绘制场景内容。此时都绘制到了帧缓存的纹理上。
3. 再绑定默认的帧缓存来激活它。
4. 使用上面的纹理来绘制四边形，同时关闭深度测试（因为绘制一个四边形不需要深度测试）。

最后，我们能看到场景内容，结果和之前不使用自定义的帧缓存是一样的。然而这有什么好处呢？那就是场景中的任何像素已经被当作一个纹理图像了，我们可以在片段着色器中对其创建一些有意思的效果。所有这些有意思的效果统称为后处理特效。

---
> 参考
[LearnOpenGL-CN 帧缓冲](http://learnopengl-cn.readthedocs.org/zh/latest/04%20Advanced%20OpenGL/05%20Framebuffers/)