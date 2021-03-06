---
layout: post
title: OpenGL 片元的测试流程
description: "OpenGL片元测试流程简介"
modified: 2015-10-20
tags: [OpenGL]
---

## 片元的测试与操作

当OpenGL在执行片断着色器内容后，会经过几个处理阶段，判断片元是否可以作为像素绘制到缓存中，以及控制绘制的方式。

比如，如果片元超出了帧缓存的矩形区域，或者它与当前帧缓存中同位置的像素相比，距离视点更远，那么正在处理的过程都会停止，片元也不会被绘制。而另一个阶段当中，片元的颜色会与当前帧缓存中的像素颜色进行混合。

片元在进入到帧缓存之需要经过许多测试过程，并且写入时可以执行一些操作。这些测试和操作大部分都可以通过glEnable()和glDisable()来分别启用和禁止。如果一个片元在某个测试过程中丢弃，那么之后所有的测试或者操作都不会再执行。这些测试和操作的发生顺序如下所示：

1. 剪切测试（scissor test）
2. 多重采样的片元操作。
3. 模板测试（stencil test）
4. 深度测试（depth test）
5. 混合（blending）
6. 抖动（dithering）
7. 逻辑操作


## 剪切测试
我们将程序窗口中的一个矩形区域称作剪切盒（scissorbox），并且将所有的绘制操作都限制在这个区域内。
我们可以使用glScissor()来设置这个剪切盒，并且使用glEnable()开启测试。如果片元位于矩形区域内，那么它将通过剪切测试。
默认条件下，剪切矩形与窗口的大小是相等的，并且剪切测试是关闭的。如果已经开启测试，那么所有的渲染，包括窗口的清除，都被限制在剪切盒区域内（这一点与视口的设置不同，后者不会限制屏幕的清除操作）。

## 多重采样的片元操作
多重采样（multisampling）是一种对集合图元的边缘进行平滑的技术 - 通常也成为反走样（antialiasing）。
默认情况下，多重采样在计算片元的覆盖率时不会考虑alpha的影响。不过，如果使用glEnable()开启某个特定模式，那么片元的alpha值将被纳入到计算过程中。

## 模板测试
模板缓存的用途之一，就是将绘图范围限制在屏幕的特定区域。模板缓存用来进行复杂的掩模(masking)操作。

## 深度测试
深度其实就是该象素点在3d世界中距离摄象机的距离。深度测试决定了是否绘制较远的象素点（或较近的象素点）。

## 混合
如果一个输入的片元通过了所有测试，那么它就可以与颜色缓存中当前的内容通过某种方式进行合并了。

## 抖动
对于颜色位面数目较小的系统来说，我们可以通过对图像中的颜色进行抖动(dithering)来提升颜色的分辨率，代价是损失一定的空同分辨率。
抖动操作本身是与硬件相关的。OpenGL能做的只是允许开启或者关团这个特性。事实上，在某些机器上，如果颜色分辨率已经非常高，那么开启抖动可能不会产生任何效果。如果要开启或者关团抖动的特性，我们以将参数GL_DITHER传人glEnable()和glDisable()。默认情况下抖动是开启的。

## 逻辑操作
片元的最后一个操作就是逻辑操作。包括或（OR）、异或(XOR)和反转(INVERT)，它作用于输人的片元数据（源）以及当前颜色缓存中的数据（目标）。这类片元操作对于位块传输（bit-blt）类型的系统是非常有用的，因为对它们来说，主要的图形操作就是将窗口中的某一处矩形数据拷贝到另外一处，或者从窗口拷贝到处理器内存，以及从内存拷贝到窗口。通常情况下，这一步拷贝操作不会将数据直接写入内存上，而是允许用户对输入的数据和己有数据之间做一次逻辑操作，然后用操作的结果替换当前已有的数据。
由于这个过程的实现代价对于硬件来说是非常低廉的，因此很多系统都允许这种做法。我们以异或(XOR)操作为例，它可以用来实现可逆的图像绘制操作，因为只要第二次使用XOR进行绘制，就可以还原原始图像。
我们可以将GL_COLOR_LOGIC_OP参数传递给glEnabIe()和glDisable()来开启和禁用逻辑搡作，否则它将保持默认的状态值，也就是GL_COPY。