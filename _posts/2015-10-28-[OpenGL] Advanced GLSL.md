---
layout: post
title: OpenGL 高级GLSL
description: "OpenGL 高级GLSL 简介"
modified: 2015-10-28
tags: [OpenGL]
---

我们会讨论一些内建变量、组织着色器输入和输出的新方式以及一个叫做uniform缓冲对象的非常有用的工具。

##GLSL的内建变量：
着色器是很小的，如果我们需要从当前着色器以外的别的资源里的数据，那么我们就不得不传给它。我们学过了使用顶点属性(vertex attributes)、uniform和采样器(samplers)可以实现这个目标。GLSL有几个以gl_为前缀的变量，使我们有一个额外的手段来获取和写入数据。其中两个我们已经打过交道了：gl_Position和gl_FragCoord，前一个是顶点着色器的输出向量，后一个是片段着色器的。

下面介绍几个有趣的GLSL内建变量，如果你想看到所有的内建变量最好去查看OpenGL的[wiki](http://www.opengl.org/wiki/Built-in_Variable_(GLSL))。

###顶点着色器变量
	
####gl_Position
我们已经了解gl_Position是顶点着色器裁切空间输出的位置向量。如果你想让屏幕上渲染出东西gl_Position必须被使用。否则我们什么都看不到。
	
####gl_PointSize
我们可以使用的一种基本图形(primitive)是GL_POINTS，每个顶点(vertex)作为一个基本图形，被渲染为一个点(point)。可以使用glPointSize函数来设置这个点的大小，但我们还可以在顶点着色器里修改点的大小。
	
GLSL有一个输出变量叫做gl_PointSize，它是一个float变量，你可以以像素的方式设置点的高度和宽度。它在着色器中描述每个顶点做为点被绘制出来的大小。
	
在着色器中影响点的大小默认是关闭的，但是如果你打算开启它，你需要开启OpenGL的GL_PROGRAM_POINT_SIZE：	glEnable(GL_PROGRAM_POINT_SIZE);
	
想象一下，每个顶点表示出来的点的大小的不同，如果用在像粒子生成之类的技术里会挺有意思的。
	
####gl_VertexID
gl_Position和gl_PointSize都是输出变量，它们的值是作为顶点着色器的输出被读取的；通过向它们写入数据可以影响结果。
顶点着色器也为我们提供了一个有趣的输入变量，我们只能从它那里读取，这个变量叫做gl_VertexID。
	
gl_VertexID是个整型变量，它储存着我们绘制的当前顶点的ID。当进行索引渲染（indexed rendering，使用glDrawElements渲染）时，这个变量保存着当前绘制的顶点的索引。当用的不是索引绘制（glDrawArrays）时，这个变量保存的是从渲染开始起直到当前处理顶点的编号。
	
###片段着色器的变量
	
####gl_FragCoord	
它是片段着色器的输入变量，类型为vec4向量。
在讨论深度测试时，我们已经了解到，gl_FragCoord向量的z元素和特定的fragment的深度值相等。然而，我们也可以使用这个向量的x和y元素来实现一些有趣的效果。
	
gl_FragCoord的x和y元素是当前片段的窗口空间坐标（window-space coordinate）。它们的起始处是窗口的左下角。如果我们的窗口是800×600的，那么一个片段的窗口空间坐标x的范围就在0到800之间，y在0到600之间。
	
我们可以使用片段着色器基于片段的窗口坐标计算出一个不同的颜色。gl_FragCoord变量的一个常用的方式是与一个不同的片段计算出来的视频输出进行对比，通常在技术演示中常见。比如我们可以把屏幕分为两个部分，窗口的左侧渲染一个输出，窗口的右边渲染另一个输出。下面是一个基于片段的窗口坐标的位置的不同输出不同的颜色的片段着色器：

	void main()
	{
	    if(gl_FragCoord.x < 400)
	        color = vec4(1.0f, 0.0f, 0.0f, 1.0f);
	    else
	        color = vec4(0.0f, 1.0f, 0.0f, 1.0f);
	}
	
####gl_FrontFacing
片段着色器另一个有意思的输入变量是gl_FrontFacing变量。在面剔除教程中，我们提到过OpenGL可以根据顶点绘制顺序弄清楚一个面是正面还是背面。gl_FrontFacing变量能告诉我们当前三角形图元是正面的还是背面的。然后我们可以决定做一些事情，比如为正面计算出不同的颜色。
	
gl_FrontFacing变量是一个布尔值，如果当前片段是正面的一部分那么就是true，否则就是false。这样我们可以创建一个立方体，里面和外面使用不同的纹理：
	
	#version 330 core
	out vec4 color;
	in vec2 TexCoords;
	
	uniform sampler2D frontTexture;
	uniform sampler2D backTexture;
	
	void main()
	{
	    if(gl_FrontFacing)
	        color = texture(frontTexture, TexCoords);
	    else
	        color = texture(backTexture, TexCoords);
	}
	
注意，如果你开启了面剔除，你就看不到箱子里面有任何东西了，所以此时使用gl_FrontFacing就没有意义了。

####gl_FragDepth
输入变量gl_FragCoord让我们可以读得当前片段的窗口空间坐标和深度值，但是它是只读的。我们不能影响到这个片段的窗口屏幕坐标，但是可以设置这个像素的深度值。GLSL给我们提供了一个叫做gl_FragDepth的变量，我们可以用它在着色器中设置像素的深度值。
	
	gl_FragDepth = 0.0f; //现在片段的深度值被设为0
	
如果着色器中没有向gl_FragDepth写入值，它就会自动采用gl_FragCoord.z的值。
	
我们自己设置深度值有一个显著缺点，因为只要我们在片段着色器中对gl_FragDepth写入什么，OpenGL就会关闭所有的前置深度测试。它被关闭的原因是，在我们运行片段着色器之前OpenGL搞不清像素的深度值，因为片段着色器可能会完全改变这个深度值。
	
因此，你需要考虑到gl_FragDepth写入所带来的性能的下降。然而从OpenGL4.2起，我们仍然可以对二者进行一定的调和，这需要在片段着色器的顶部使用深度条件（depth condition）来重新声明gl_FragDepth：
	
	layout (depth_<condition>) out float gl_FragDepth;
	
condition可以使用下面的值：
	
|Condition  |   描述        |
|:-------:  |:-------:      |
|any        |	默认值. 前置深度测试是关闭的，你失去了很多性能表现|
|----
|greater    |	深度值只能比gl_FragCoord.z大|
|----
|less	    |     深度值只能设置得比gl_FragCoord.z小|
|----
|unchanged  |	如果写入gl_FragDepth, 你就会写gl_FragCoord.z|
|====
{: rules="groups"}
    
下面是一个在片段着色器里增加深度值的例子，不过仍可开启前置深度测试：
	
	#version 330 core
	layout (depth_greater) out float gl_FragDepth;
	out vec4 color;
	
	void main()
	{
	    color = vec4(1.0f);
	    gl_FragDepth = gl_FragCoord.z + 0.1f;
	}
	
这个功能只在OpenGL4.2以上版本才有。
	
	
##接口块(Interface blocks)：
到目前位置，每次我们打算从顶点向片段着色器发送数据，我们都会声明一个相互匹配的输出/输入变量。从一个着色器向另一个着色器发送数据，一次将它们声明好是最简单的方式，但是随着应用变得越来越大，你也许会打算发送的不仅仅是变量，最好还可以包括数组和结构体。

为了帮助我们组织这些变量，GLSL为我们提供了一些叫做接口块(Interface blocks)的东西，好让我们能够组织这些变量。声明接口块和声明struct有点像，不同之处是它现在基于块（block），使用in和out关键字来声明，最后它将成为一个输入或输出块（block）。

    #version 330 core
    layout (location = 0) in vec3 position;
    layout (location = 1) in vec2 texCoords;

    uniform mat4 model;
    uniform mat4 view;
    uniform mat4 projection;

    out VS_OUT
    {
        vec2 TexCoords;
    } vs_out;

    void main()
    {
        gl_Position = projection * view * model * vec4(position, 1.0f);
        vs_out.TexCoords = texCoords;
    }

这次我们声明一个叫做vs_out的接口块，它把我们需要发送给下个阶段着色器的所有输出变量组合起来。当我们希望把着色器的输入和输出组织成数组的时候它就变得很有用。

然后，我们还需要在下一个着色器——片段着色器中声明一个输入interface block。块名（block name）应该是一样的，但是实例名可以是任意的。

    #version 330 core
    out vec4 color;

    in VS_OUT
    {
        vec2 TexCoords;
    } fs_in;

    uniform sampler2D texture;

    void main()
    {
        color = texture(texture, fs_in.TexCoords);
    }

如果两个interface block名一致，它们对应的输入和输出就会匹配起来。这是另一个可以帮助我们组织代码的有用功能，特别是在跨着色阶段的情况，比如几何着色器。

##uniform缓冲对象 (Uniform buffer objects)：
目前为止，当我们时用一个以上的着色器的时候，我们必须一次次设置uniform变量，尽管对于每个着色器来说它们都是一样的。所以为什么还麻烦地多次设置它们呢？

OpenGL为我们提供了一个叫做uniform缓冲对象的工具，使我们能够声明一系列的全局uniform变量， 它们会在几个着色器程序中保持一致。当使用uniform缓冲的对象时相关的uniform只能设置一次。我们仍需为每个着色器手工设置唯一的uniform。创建和配置一个uniform缓冲对象需要费点功夫。

因为uniform缓冲对象是一个缓冲，因此我们可以使用glGenBuffers创建一个，然后绑定到GL_UNIFORM_BUFFER缓冲目标上，然后把所有相关uniform数据存入缓冲。有一些原则，像uniform缓冲对象如何储存数据，我们会在稍后讨论。首先我们我们在一个简单的顶点着色器中，用uniform块(uniform block)储存投影和视图矩阵：

    #version 330 core
    layout (location = 0) in vec3 position;

    layout (std140) uniform Matrices
    {
        mat4 projection;
        mat4 view;
    };

    uniform mat4 model;

    void main()
    {
        gl_Position = projection * view * model * vec4(position, 1.0);
    }
    
前面，大多数例子里我们在每次渲染迭代，都为projection和view矩阵设置uniform。这个例子里使用了uniform缓冲对象，这非常有用，因为这些矩阵我们设置一次就行了。

在这里我们声明了一个叫做Matrices的uniform块，它储存两个4×4矩阵。在uniform块中的变量可以直接获取，而不用使用block名作为前缀。接着我们在缓冲中储存这些矩阵的值，每个声明了这个uniform块的着色器都能够获取矩阵。

现在你可能会奇怪layout(std140)是什么意思。它的意思是说当前定义的uniform块为它的内容使用特定的内存布局，这个声明实际上是设置uniform块布局(uniform block layout)。

##uniform块布局(uniform block layout)：
一个uniform块的内容被储存到一个缓冲对象中，实际上就是在一块内存中。因为这块内存也不清楚它保存着什么类型的数据，我们就必须告诉OpenGL哪一块内存对应着色器中哪一个uniform变量。

假想下面的uniform块在一个着色器中：

    layout (std140) uniform ExampleBlock
    {
        float value;
        vec3 vector;
        mat4 matrix;
        float values[3];
        bool boolean;
        int integer;
    };

我们所希望知道的是每个变量的大小（以字节为单位）和偏移量（从block的起始处），所以我们可以以各自的顺序把它们放进一个缓冲里。每个元素的大小在OpenGL中都很清楚，直接与C++数据类型呼应，向量和矩阵是一个float序列（数组）。OpenGL没有澄清的是变量之间的间距。这让硬件以它认为合适的位置放置变量。

GLSL 默认使用的uniform内存布局叫做共享布局(shared layout)，因为一旦偏移量被硬件定义，它们就会持续地被多个程序所共享。使用共享布局，GLSL可以为了优化而重新放置uniform变量，只要变量的顺序保持不变。

尽管共享布局给我们做了一些节约空间的优化，但通常在实践中并不适用分享布局，而是使用std140布局。std140通过一系列规范声明了它们各自的偏移量，为每个变量类型显式地声明了内存的布局。由于被显式的提及，我们就可以手工算出每个变量的偏移量。

每个变量都有一个基线对齐(base alignment)，它等于在一个uniform块中这个变量所占的空间（包含边距），这个基线对齐是使用std140布局原则计算出来的。然后，我们为每个变量计算出它的对齐偏移(aligned offset)，这是一个变量从块（block）开始处的字节偏移量。变量对齐的字节偏移一定等于它的基线对齐的倍数。

准确的布局规则可以在OpenGL的uniform缓冲规范中得到，但我们会列出最常见的规范。GLSL中每个变量类型比如int、float和bool被定义为4字节，每4字节被表示为N。

|类型	        |布局规范       |
|:-------:    |:-------:      |
|像int和bool这样的标量	| 每个标量的基线为N|
|----
|向量	        |每个向量的基线是2N或4N大小。这意味着vec3的基线为4N|
|----
|标量与向量数组 |	每个元素的基线与vec4的相同|
|----
|矩阵         |	被看做是存储着大量向量的数组，每个元素的基数与vec4相同|
|----
|结构体        |	根据以上规则计算其各个元素，并且间距必须是vec4基线的倍数|
|====

再次利用之前介绍的uniform块ExampleBlock，我们用std140布局，计算它的每个成员的aligned offset（对齐偏移）：

    layout (std140) uniform ExampleBlock
    {
                            // base alignment ----------    // aligned offset
        float value;        // 4                            // 0
        vec3 vector;        // 16                           // 16 (必须是16的倍数，因此 4->16)
        mat4 matrix;        // 16                           // 32  (第 0 行)
                            // 16                           // 48  (第 1 行)
                            // 16                           // 64  (第 2 行)
                            // 16                           // 80  (第 3 行)
        float values[3];    // 16 (数组中的标量与vec4相同)    // 96 (values[0])
                            // 16                           // 112 (values[1])
                            // 16                           // 128 (values[2])
        bool boolean;       // 4                            // 144
        int integer;        // 4                            // 148
    };

使用计算出来的偏移量，根据std140布局规则，我们可以用glBufferSubData这样的函数，使用变量数据填充缓冲。虽然不是很高效，但std140布局可以保证在每个程序中声明的这个uniform块的布局保持一致。

在定义uniform块前面添加layout (std140)声明，我们就能告诉OpenGL这个uniform块使用了std140布局。另外还有两种其他的布局可以选择，它们需要我们在填充缓冲之前查询每个偏移量。我们已经了解了分享布局（shared layout）和其他的布局都将被封装（packed）。当使用封装（packed）布局的时候，不能保证布局在别的程序中能够保持一致，因为它允许编译器从uniform块中优化出去uniform变量，这在每个着色器中都可能不同。


##使用uniform缓冲：
我们讨论了uniform块在着色器中的定义和如何定义它们的内存布局，但是我们还没有讨论如何使用它们。

首先我们需要创建一个uniform缓冲对象，这要使用glGenBuffers来完成。当我们拥有了一个缓冲对象，我们就把它绑定到GL_UNIFORM_BUFFER目标上，调用glBufferData来给它分配足够的空间。

    GLuint uboExampleBlock;
    glGenBuffers(1, &uboExampleBlock);
    glBindBuffer(GL_UNIFORM_BUFFER, uboExampleBlock);
    // 分配150个字节的内存空间
    glBufferData(GL_UNIFORM_BUFFER, 150, NULL, GL_STATIC_DRAW); 
    glBindBuffer(GL_UNIFORM_BUFFER, 0);

现在任何时候当我们打算往缓冲中更新或插入数据，我们就绑定到uboExampleBlock上，并使用glBufferSubData来更新它的内存。我们只需要更新这个uniform缓冲一次，所有的使用这个缓冲着色器就都会使用它更新的数据了。但是，OpenGL是如何知道哪个uniform缓冲对应哪个uniform块呢？

在OpenGL上下文（context）中，定义了若干绑定点（binding points），在那我们可以把一个uniform缓冲链接上去。当我们创建了一个uniform缓冲，我们把它链接到一个绑定点上，我们也把着色器中uniform块链接到同一个绑定点上，这样就把它们链接到一起了。下面的图标表示了这点：

![]({{ site.url }}/images/post/2015-10-28/OpenGL GLSL 1.png)

你可以看到，我们可以将多个uniform缓冲绑定到不同绑定点上。因为着色器A和着色器B都有一个链接到同一个绑定点0的uniform块，它们的uniform块分享同样的uniform数据—uboMatrices。有一个前提条件是两个着色器必须都定义了Matrices这个uniform块。

我们调用glUniformBlockBinding函数来把uniform块设置到一个特定的绑定点上。函数的第一个参数是一个程序对象，接着是一个uniform块索引（uniform block index）和打算链接的绑定点。uniform块索引是一个着色器中定义的uniform块的索引位置，可以调用glGetUniformBlockIndex来获取这个值，这个函数接收一个程序对象和uniform块的名字。我们可以从图表设置Lights这个uniform块链接到绑定点2：

    GLuint lights_index = glGetUniformBlockIndex(shaderA.Program, "Lights");
    glUniformBlockBinding(shaderA.Program, lights_index, 2);

注意，我们必须在每个着色器中重复做这件事。

从OpenGL4.2起，也可以在着色器中通过添加另一个布局标识符来储存一个uniform块的绑定点，就不用我们调用glGetUniformBlockIndex和glUniformBlockBinding了。下面显式设置了Lights这个uniform块的绑定点：

    layout(std140, binding = 2) uniform Lights { ... };

然后我们还需要把uniform缓冲对象绑定到同样的绑定点上，这个可以使用glBindBufferBase或glBindBufferRange来完成。

    glBindBufferBase(GL_UNIFORM_BUFFER, 2, uboExampleBlock);
    // 或者
    glBindBufferRange(GL_UNIFORM_BUFFER, 2, uboExampleBlock, 0, 150);

函数glBindBufferBase接收一个目标、一个绑定点索引和一个uniform缓冲对象作为它的参数。这个函数把uboExampleBlock链接到绑定点2上面，自此绑定点所链接的两端都链接在一起了。你还可以使用glBindBufferRange函数，这个函数还需要一个偏移量和大小作为参数，这样你就可以只把一定范围的uniform缓冲绑定到一个绑定点上了。使用glBindBufferRage函数，你能够将多个不同的uniform块链接到同一个uniform缓冲对象上。

现在所有事情都做好了，我们可以开始向uniform缓冲添加数据了。我们可以使用glBufferSubData将所有数据添加为一个单独的字节数组或者更新缓冲的部分内容，只要我们愿意。为了更新uniform变量boolean，我们可以这样更新uniform缓冲对象：

    glBindBuffer(GL_UNIFORM_BUFFER, uboExampleBlock);
    GLint b = true; // GLSL中的布尔值是4个字节，因此我们将它创建为一个4字节的整数
    glBufferSubData(GL_UNIFORM_BUFFER, 144, 4, &b);
    glBindBuffer(GL_UNIFORM_BUFFER, 0);
    
同样的处理也能够应用到uniform块中其他uniform变量上。


uniform缓冲对象比单独的uniform有很多好处。
第一，一次设置多个uniform比一次设置一个速度快。
第二，如果你打算改变一个横跨多个着色器的uniform，在uniform缓冲中只需更改一次。
最后一个好处可能不是很明显，使用uniform缓冲对象你可以在着色器中使用更多的uniform。OpenGL有一个对可使用uniform数据的数量的限制，可以用GL_MAX_VERTEX_UNIFORM_COMPONENTS来获取。当使用uniform缓冲对象中，这个限制的阈限会更高。所以无论何时，你达到了uniform的最大使用数量（比如做骨骼动画的时候），你可以使用uniform缓冲对象。
