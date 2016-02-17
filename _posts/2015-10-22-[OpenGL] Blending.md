---
layout: post
title: OpenGL 混合（Blending）
description: "OpenGL混合简介"
modified: 2015-10-22
tags: [OpenGL]
---

## 透明
如果一个输入的片元通过了所有相关的片元测试，那么它就可以与颜色缓存中的内容以某种方式进行合并了。最简单的，也是默认的方式，就是直接覆盖已有的值，实际上这样不能称作是合并。除此之外，我们也可以将帧缓存中已有的颜色与输入的片元颜色进行混合（blending）。

大多数情况下，混合是与片元的alpha值直接相关的，不过这也并不是一个硬性的要求。alpha是颜色的第四个分量，OpenGL中的所有颜色都会带有alpha值（无论你是否显式地设置了它）。它是透明度的一种度量方式，我们可以用它来实现各种半透明物体的模拟。

对物体来说透明并不是纯色而是混合色，因为这种颜色来自于不同浓度的自身颜色和它后面的物体颜色。如果需要在OpenGL中使用alpha值，那么管线就需要得到更多有关当前图元颜色（也就是片元着色器输出的颜色值）的信息。

透明物体可以是完全透明（它使颜色完全穿透）或者半透明的（它使颜色穿透的同时也显示自身颜色）。我们之前所使用的纹理都是由RGB这3个元素组成的，但是有些纹理同样有一个内嵌的alpha通道，它为每个纹理像素（Texel）包含着一个alpha值。这个alpha值告诉我们纹理的哪个部分有透明度，以及这个透明度有多少。


## 忽略片段
有些图像并不关心半透明度，但也想基于纹理的颜色值显示一部分。例如，创建草只需要把一个草的纹理贴到2D四边形上，然后把这个四边形放置到你的场景中。
对于这样一种alpha值只有0和1的图片，我们要忽略纹理透明部分的像素，不必将这些片段储存到颜色缓冲中。
我们可以在片段着色器中这样设置：

    if(texColor.a < 0.1)  discard;

我们检查被采样纹理颜色是否包含一个低于0.1的alpha值，如果有，就丢弃这个片段。


## 混合
上述丢弃片段的方式，不能使我们渲染半透明图像，我们要么渲染出像素，要么完全地丢弃它。为了渲染出不同的透明度级别，我们需要开启混合(Blending)。像大多数OpenGL的功能一样，我们可以开启GL_BLEND来启用混合功能： glEnable(GL_BLEND);

开启混合后，我们还需要告诉OpenGL它该如何混合。
OpenGL以下面的方程进行混合：

![]({{ site.url }}/images/post/2015-10-22/OpenGL Blending 1.png)

- C¯source：源颜色向量。这是来自纹理的本来的颜色向量。
- C¯destination：目标颜色向量。这是储存在颜色缓冲中当前位置的颜色向量。
- Fsource：源参数。设置了对源颜色的alpha值影响。
- Fdestination：目标参数。设置了对目标颜色的alpha影响。

片段着色器运行完成并且所有的测试都通过以后，混合方程才能自由执行片段的颜色输出，当前它在颜色缓冲中（前面片段的颜色在当前片段之前储存）。源和目标颜色会自动被OpenGL设置，而源和目标参数可以让我们自由设置。我们来看一个简单的例子：

![]({{ site.url }}/images/post/2015-10-22/OpenGL Blending 2.png)

我们有两个方块，我们希望在红色方块上绘制绿色方块。红色方块会成为源颜色（它会先进入颜色缓冲），我们将在红色方块上绘制绿色方块。

那么怎样来设置参数呢？我们把Fsource设置为源颜色向量的alpha值：0.6。接着，让目标方块的浓度等于剩下的alpha值。方程将变成：

![]({{ site.url }}/images/post/2015-10-22/OpenGL Blending 3.png)

![]({{ site.url }}/images/post/2015-10-22/OpenGL Blending 4.png)

最终方块结合部分包含了60%的绿色和40%的红色。最后的颜色被储存到颜色缓冲中，取代先前的颜色。

使用glBlendFunc的函数可以设置参数。

    void glBlendFunc(GLenum sfactor, GLenum dfactor)

两个参数，来设置源（source）和目标（destination）参数。
![glBlendFunc 参数]({{ site.url }}/images/post/2015-10-22/OpenGL Blending 5.png)

上面的例子，我们打算把源颜色的alpha给源因子，1-alpha给目标因子。调整到glBlendFunc之后就像这样：

    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);


也可以使用glBlendFuncSeparate函数，为RGB和alpha通道各自设置不同的选项：

    glBlendFuncSeparate(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA, GL_ONE,  GL_ZERO);


现在，源和目标元素已经相加了。如果我们愿意的话，我们还可以把它们相减。
void glBlendEquation(GLenum mode)允许我们设置这个操作，有3种可行的选项：

- GL_FUNC_ADD：默认的，彼此元素相加：C¯result = Src + Dst.
- GL_FUNC_SUBTRACT：彼此元素相减： C¯result = Src – Dst.
- GL_FUNC_REVERSE_SUBTRACT：彼此元素相减，但顺序相反：C¯result = Dst – Src.

通常我们可以省略glBlendEquation因为GL_FUNC_ADD在大多数时候就是我们想要的。


## 渲染半透明纹理
此时我们只要开启深度测试，并设置合适的混合方程：

    glEnable(GL_BLEND);
    glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);

![]({{ site.url }}/images/post/2015-10-22/OpenGL Blending 6.png)

在绘制一系列半透明窗子时，出现了问题，前面的窗子透明部分阻塞了后面的。
这是因为深度测试在与混合一同工作时出现了点状况。当写入深度缓冲的时候，深度测试不关心片段是否有透明度，所以透明部分被写入深度缓冲，就和其他值没什么区别。结果是整个四边形的窗子被检查时都忽视了透明度。即便透明部分应该显示出后面的窗子，深度缓冲还是丢弃了它们。

要让混合在多物体上有效，我们必须先绘制最远的物体，最后绘制最近的物体。普通的无混合物体仍然可以使用深度缓冲正常绘制，所以不必给它们排序。我们一定要保证它们在透明物体前绘制好。当无透明度物体和透明物体一起绘制的时候，通常要遵循以下原则：

1. 先绘制所有不透明物体。 
2. 为所有透明物体排序。 
3. 按顺序绘制透明物体。
	
一种排序透明物体的方式是，获取一个物体到观察者透视图的距离。这可以通过获取摄像机的位置向量和物体的位置向量来得到。

虽然这种按照距离对物体进行排序的方法在特定的场景中能够良好工作，但它不能进行旋转、缩放或者进行其他的变换，奇怪形状的物体需要一种不同的方式，而不能简单的使用位置向量。

在场景中排序物体是个有难度的技术，它很大程度上取决于你场景的类型，更不必说会耗费额外的处理能力了。完美地渲染带有透明和不透明的物体的场景并不那么容易。有更高级的技术例如次序无关透明度（order independent transparency）。

---
> 参考
[LearnOpenGL-CN 混合](http://learnopengl-cn.readthedocs.org/zh/latest/04%20Advanced%20OpenGL/03%20Blending/)