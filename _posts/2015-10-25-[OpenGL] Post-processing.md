---
layout: post
title: OpenGL 后处理（Post-processing）
description: "OpenGL 后处理 简介"
modified: 2015-10-25
tags: [OpenGL]
---

我们前面已经把整个场景渲染到了一个单独的纹理上，我们可以创建一些有趣的效果，只要简单操纵纹理数据就能做到。这部分，我们展示一些常用的后处理特效，以及怎样添加一些创造性去创建出你自己的特效。

##反相（Inversion）：
我们已经取得了渲染输出的每个颜色，所以在片段着色器里返回这些颜色的反色并不难。我们得到屏幕纹理的颜色，然后用1.0减去它：

    void main()
    {
        color = vec4(vec3(1.0 - texture(screenTexture, TexCoords)), 1.0);
    }

![反相效果]({{ site.url }}/images/post/2015-10-25/OpenGL Post 1.png)

##灰度（Grayscale）：
另一个有意思的效果是移除所有除了黑白灰以外的颜色作用，使整个图像成为黑白的。实现它的简单的方式是获得所有颜色元素，然后将它们平均化（因为当RGB值是相等时，就是灰度图）：

    void main()
    {
        color = texture(screenTexture, TexCoords);
        float average = (color.r + color.g + color.b) / 3.0;
        color = vec4(average, average, average, 1.0);
    }

这已经创造出很赞的效果了，但是人眼趋向于对绿色更敏感，对蓝色感知比较弱，所以为了获得更精确的符合人体物理的结果，我们需要使用加权通道（weighted channels）：

    void main()
    {
        color = texture(screenTexture, TexCoords);
        float average = 0.2126 * color.r + 0.7152 * color.g + 0.0722 * color.b;
        color = vec4(average, average, average, 1.0);
    }

![灰度效果]({{ site.url }}/images/post/2015-10-25/OpenGL Post 2.png)

##Kernel effects:
在单独纹理图像上进行后处理的另一个好处是我们可以从纹理的其他部分进行采样。比如我们可以从当前纹理坐标的周围采样多个纹理值。创造性地把它们结合起来就能创造出有趣的效果了。

kernel是一个长得有点像一个小矩阵的数值数组，它中间的值中心可以映射到一个像素上，这个像素和这个像素周围的值再乘以kernel，最后再把结果相加就能得到一个值。所以，我们基本上就是给当前纹理坐标加上一个它四周的偏移量，然后基于kernel把它们结合起来。下面是一个kernel的例子：

![kernel数组的例子]({{ site.url }}/images/post/2015-10-25/OpenGL Post 3.png)

这个kernel表示一个像素周围八个像素乘以2，它自己乘以-15。这个例子基本上就是把周围像素乘上2，中间像素去乘以一个比较大的负数来进行平衡。

注：你在网上能找到的kernel的例子大多数都是所有值加起来等于1，如果加起来不等于1就意味着这个纹理值比原来更大或者更小了。

kernel对于后处理来说非常管用，因为用起来简单，网上能找到有很多实例。这里假设每个kernel都是3×3（实际上大多数都是3×3）：

    const float offset = 1.0 / 300;  

    void main()
    {
        vec2 offsets[9] = vec2[](
            vec2(-offset, offset),  // top-left
            vec2(0.0f,    offset),  // top-center
            vec2(offset,  offset),  // top-right
            vec2(-offset, 0.0f),    // center-left
            vec2(0.0f,    0.0f),    // center-center
            vec2(offset,  0.0f),    // center-right
            vec2(-offset, -offset), // bottom-left
            vec2(0.0f,    -offset), // bottom-center
            vec2(offset,  -offset)  // bottom-right
        );

        float kernel[9] = float[](
            -1, -1, -1,
            -1,  9, -1,
            -1, -1, -1
        );

        vec3 sampleTex[9];
        for(int i = 0; i < 9; i++)
        {
            sampleTex[i] = vec3(texture(screenTexture, TexCoords.st + offsets[i]));
        }
        vec3 col;
        for(int i = 0; i < 9; i++)
            col += sampleTex[i] * kernel[i];

        color = vec4(col, 1.0);
    }

在片段着色器中我们先为每个四周的纹理坐标创建一个9个vec2偏移量的数组。偏移量是一个简单的常数，你可以设置为自己喜欢的。接着我们定义kernel，这里应该是一个锐化kernel，它通过一种有趣的方式从所有周边的像素采样，对每个颜色值进行锐化。最后，在采样的时候我们把每个偏移量加到当前纹理坐标上，然后用加在一起的kernel的值乘以这些纹理值。

![kernel 锐化效果]({{ site.url }}/images/post/2015-10-25/OpenGL Post 4.png)

##Blur：
创建模糊效果的kernel定义如下：

![模糊效果 kernel]({{ site.url }}/images/post/2015-10-25/OpenGL Post 5.png)

由于所有数值加起来的总和为16,简单返回结合起来的采样颜色是非常亮的,所以我们必须将kernel的每个值除以16。最终的kernel数组会是这样的:

    float kernel[9] = float[](
        1.0 / 16, 2.0 / 16, 1.0 / 16,
        2.0 / 16, 4.0 / 16, 2.0 / 16,
        1.0 / 16, 2.0 / 16, 1.0 / 16  
    );

我们可以随着时间的变化改变模糊量，创建出类似于某人喝醉酒的效果。或者，当我们的主角摘掉眼镜的时候增加模糊。模糊也能为我们在后面的教程中提供对颜色值进行平滑处理的能力。

你可以看到我们一旦拥有了这个kernel的实现以后,创建一个后处理特效就不再是一件难事。

![kernel 模糊效果]({{ site.url }}/images/post/2015-10-25/OpenGL Post 6.png)

##边检测：

![边检测 kernel]({{ site.url }}/images/post/2015-10-25/OpenGL Post 7.png)

这个kernel将所有的边提高亮度，而对其他部分进行暗化处理，当我们值关心一副图像的边缘的时候，它非常有用。

![边检测效果]({{ site.url }}/images/post/2015-10-25/OpenGL Post 8.png)


---
> 参考
[LearnOpenGL-CN 帧缓冲](http://learnopengl-cn.readthedocs.org/zh/latest/04%20Advanced%20OpenGL/05%20Framebuffers/)