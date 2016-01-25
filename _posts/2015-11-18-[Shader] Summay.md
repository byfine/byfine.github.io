---
layout: post
title: Shader：概述 
description: "Shader概述"
modified: 2015-11-18
tags: [Shader]
---

##GPU发展
GPU英文全称Graphic Processing Unit，中文翻译为“图形处理器”。NVIDIA在1999年发布GeForce 256图形处理芯片时首先提出GPU的概念。GPU所采用的核心技术有硬件T&L、立方纹理(Cube map)和顶点混合、纹理压缩和凹凸映射贴图、双重纹理四像素256位渲染引擎等，而硬件T&L（Transform and Lighting，多边形转换与光源处理）技术可以说是GPU的标志。 
2001年第三代modern GPU提供了vertex programmability（顶点编程能力），允许应用程序指定一个序列的进行定点操作控制。所谓vertex，就是组成3D图形的顶点，vertex信息包含了3D模型在空间内的坐标等信息，vertex shader可以通过特定的算法在工作中改变3D模型的外形。到了2003年，GPU开始同时支持vertex programmability 和 fragment programmability（片段编程能力，也成像素编程）。同时DirectX 和 OpenGL也得到了发展。

##什么是Shader
Shader，即着色器。是一些在GPU上运行的小程序，可编程图形管线的算法片段，用以告诉图形硬件如何计算和输出图像。
主要分为两类：Vertex Shader 和 Fragment Shader。

##渲染管线
> 渲染管线也称为渲染流水线或像素流水线或像素管线，是[显示芯片](http://baike.baidu.com/view/7254.htm)内部处理图形信号相互独立的的[并行处理](http://baike.baidu.com/view/494465.htm)单元。在某种程度上可以把渲染管线比喻为工厂里面常见的各种生产流水线，工厂里的[生产流水线](http://baike.baidu.com/subview/803250/803250.htm)是为了提高产品的生产能力和效率，而渲染管线则是提高显卡的工作能力和效率。

##光栅化
> 光栅化就是把顶点数据转换为片元的过程。片元中的每一个元素对应于帧缓冲区中的一个像素。
光栅化其实是一种将几何图元变为二维图像的过程。该过程包含了两部分的工作。第一部分工作：决定窗口坐标中的哪些整型栅格区域被基本图元占用；第二部分工作：分配一个颜色值和一个深度值到各个区域。光栅化过程产生的是片元。
把物体的数学描述以及与物体相关的颜色信息转换为屏幕上用于对应位置的像素及用于填充像素的颜色，这个过程称为光栅化，这是一个将离散信号转换为模拟信号的过程。

渲染管线：
![渲染管线]({{ site.url }}/images/post/Shader/RenderPipeline.png)

unity图形处理流程：
![unity图形处理流程]({{ site.url }}/images/post/Shader/unity-graphics-processing.png)

##Shader、材质和贴图
输入的贴图或者颜色，经过Shaer将其与顶点数据以一定方式组合起来，然后输出。绘图单位可以依据这个输出来将图像绘制到屏幕上。输入的贴图或者颜色等，加上对应的shader，以及对shader的特定的参数设置，将这些内容（shader以及输入参数）打包存储在一起，得到的就是一个material（材质）。之后，我们便可以将材质赋予三维物体来进行渲染（输出）了。

##GLSL/HLSL/Cg
Shader language 目前主要有3种：GLSL（OpenGL Shder Language, 基于OpenGL）, HLSL（High Level Shading Language, 基于DirectX）, 还有NVIDIA公司的Cg语言（C for Graphic）。

OpenGL 支持Windows、Linux、MacOS等许多平台，DirectX仅支持Win平台。而Cg是一个可以被OpenGL和Direct3D广泛支持的图形处理器编程语言。而且Cg语言是Microsoft和NVIDIA相互协作在标准硬件光照语言的语法和语义上达成了一致而开发，所以，HLSL和Cg其实是同一种语言。

所以尽量选择Cg比较好。

##总结
一般来说，着色器会分为顶点(Vertex)着色器和片段(Fragment)着色器两个部分。

顶点着色器，主要求出各个顶点在投影面的坐标。所以传进去的值需要有顶点的三维坐标，还需要有摄像机的位移旋转矩阵。
片段着色器，主要通过给出的数据，结合从顶点着色器得到的坐标值，计算出每一个像素点应该显示的颜色值。所以传进去的值，会默认包括了顶点程序的坐标，然后我们给予的贴图信息、颜色信息和UV坐标信息。片段着色器是针对于每个像素点的，所以片段着色器的输出，就是该像素点应该显示的颜色值。

于是，我们就可以通过编写顶点着色器程序，通过一定的方式去改变顶点在投影面的坐标，让模型变形；或者通过编写片段着色器程序，让模型表现出来的颜色根据我们的需要而改变。

Shader（着色器）实际上就是一小段程序，它负责将输入的Mesh（网格）以指定的方式和输入的贴图或者颜色等组合作用，然后输出。输入的贴图或者颜色等，加上对应的Shader，以及对Shader的特定的参数设置，将这些内容（Shader及输入参数）打包存储在一起，得到的就是一个Material（材质）。

Shader开发者要做的就是根据输入，进行计算变换，产生输出而已。

##Unity Shader形态
Unity 有三种编写shader的方式：

- surface shaders
- vertex and fragment shaders
- fixed function shaders

####fixed function shader （固定功能着色器）：
对应于固定管线硬件的操作，最简单的着色器类型，只能使用Unity3D自带的固定语法和提供的方法，适用于任何硬件，使用难度最小。    

####vertex and fragment shader （顶点片段程序着色器）：
顶点和片段着色器，如前所述，是可编程图形管线主要支持的方式。是效果最为丰富的着色器类型，使用Cg/HLSL语言规范，着色器由顶点程序和片段程序组成。所有效果都需要自己编写，使用难度相对较大。

####surface shader （表面着色器）：
Unity推荐的shader类型。同样使用Cg/HLSL语言规范的着色器类型，不过把光照模型提取出来，可以使用Unity3D自带的一些光照模型，也可以自己编写光照模型，着色器同样由顶点程序和片段程序组成，不过本身有默认的程序方法，使用者可以只针对自己关系的效果部分进行编写。由于选择性比较大，所以可以编写出较为丰富的效果，使用难度相对vertex and fragment shader小。
可以理解其是对Vertex 和 Fragment shader的一种包装。
(surface shader有一个问题，它不支持SubShader内部的多pass，所以某些需要多pass的效果要实现起来会比较困难。) 

Unity建议从ShaderLab语法开始学习shader。Fixed function shader 只能被ShaderLab编写。（但是 vertex and fragment shader 和surface shader 是不限于shaderlab的，可以使用Cg/HLSL/GLSL）。

##Shaderlab基本结构

    Shader "MyShader" { 
    Properties { 
        _MyTexture ("My Texture", 2D) = "white" { } 
        //其他属性
    } 
    SubShader { 
        // - surface shader or
        // - vertex and program shader or
        // - fixed function shader 
    } 
    SubShader { 
        // 一个更简单的shader，可以运行在更弱的硬件上
    }
    Fallback "Legacy Shaders/VertexLit"
    [CustomEditor]
    }

首先是一些属性定义，属性名是前面带下划线的，显示在编辑器的名字是后面字符串中的。
接下来是一个或者多个的子着色器，只有一个能被执行，哪一个子着色器被使用是由运行的平台所决定的。子着色器是代码的主体，每一个子着色器中包含一个或者多个的Pass。
最后指定一个回滚，用来处理所有Subshader都不能运行的情况（比如目标设备实在太老，所有Subshader中都有其不支持的特性）。
CustomEditor是定制编辑器。

需要提前说明的是，在实际进行 surface shader 的开发时，我们将直接在Subshader这个层次上写代码，系统将把我们的代码编译成若干个合适的Pass。

---
> 参考：<br>
[Shader编程教程](http://edu.manew.com/course/96)<br>
[Unity Manual](http://docs.unity3d.com/Manual/SL-Reference.html)

