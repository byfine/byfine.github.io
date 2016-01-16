---
layout: post
title: OpenGL 渲染管线、VAO/VBO/EBO
description: "OpenGL渲染管线，VAO/VBO/EBO介绍"
modified: 2015-10-14
tags: [OpenGL]
---

##图形渲染管线(Pipeline)
图形渲染管线(Pipeline)，指的是一堆原始图形数据途经一个输送管道，期间经过各种变化处理最终出现在屏幕的过程。

OpenGL的大部分工作都是关于如何把3D坐标转变为适应屏幕的2D像素。这一处理过程是就由图形渲染管线管理的。它可以被划分为两个主要部分：第一个部分把你的3D坐标转换为2D坐标，第二部分是把2D坐标转变为实际的有颜色的像素。

注：2D坐标和像素也是不同的，2D坐标是在2D空间中的一个点的非常精确的表达，2D像素是这个点的近似值，它受到你的屏幕/窗口解析度的限制。

图形渲染管线可以被划分为几个阶段，每个阶段需要把前一个阶段的输出作为输入。在GPU上为每一个(渲染管线)阶段运行各自的小程序，从而在图形渲染管线中快速处理你的数据。这些小程序叫做**着色器**(Shader)。

有些着色器允许开发者自己配置，这样我们就可以更细致地控制图形渲染管线中的特定部分了，因为它们运行在GPU上，所以它们会节约宝贵的CPU时间。OpenGL着色器是用OpenGL着色器语言(OpenGL Shading Language, GLSL)写成的。

下图是一个图形渲染管线的每个阶段的抽象表达。蓝色部分代表的是我们可以自定义的着色器。
![渲染管线]({{ site.url }}/images/post/2015-10-14/OpenGL Pipline 1.png)


####顶点属性(Vertex Attributes)：
顶点数据(Vertex Data)是一些顶点的集合。一个顶点是一个3D坐标(也就是x、y、z数据)。三个3D坐标组成一个三角形。而顶点数据是用**顶点属性**(Vertex Attributes)表示的，它可以包含任何我们希望用的数据。
	
####基本图元(Primitives)：
一维或二维实体（点、线、多边形）。这些实体用来在3D空间中创建3D实体。
OpenGL需要知道我们的坐标和颜色值构成的具体是什么，点、三角形还是线？构成的这些便是基本图元，任何一个绘制命令的调用都必须把基本图形类型传递给OpenGL。这是其中的几个：GL_POINTS、GL_TRIANGLES、GL_LINE_STRIP。
	
####顶点着色器(Vertex Shader)：
顶点着色器主要的目的是把3D坐标转为另一种3D坐标(后面会解释)，同时允许我们对顶点属性进行一些基本处理。

####基本图形装配(Primitive Assembly)：
把顶点着色器的表示为基本图元的所有顶点作为输入，把所有点组装为特定的基本图元的形状。

####几何着色器(Geometry Shader)：
把基本图元形式的顶点的集合作为输入，它可以通过产生新顶点构造出新的(或是其他的)基本图元来生成其他形状。

####细分着色器(Tessellation Shaders)：
拥有把给定基本图元细分为更多小基本图形的能力。这样我们就能在物体更接近玩家的时候通过创建更多的三角形的方式创建出更加平滑的视觉效果。

####光栅化(Rasterization，也译为像素化)：
它会把基本图形映射为屏幕上相应的像素，生成供片段着色器使用的片段(Fragment)。在片段着色器运行之前，会执行裁切(Clipping)。裁切会丢弃超出你的视图以外的那些像素，来提升执行效率。

####片段着色器(Fragment Shader)：
片段着色器的主要目的是计算一个像素的最终颜色，这也是OpenGL高级效果产生的地方。通常，片段着色器包含用来计算像素最终颜色的3D场景的一些数据(比如光照、阴影、光的颜色等等)。

在所有相应颜色值确定以后，最终会进入alpha测试和混合(Blending)阶段。这个阶段检测像素的相应的深度和Stencil值，来检查这个像素是否在另一个物体的前面或后面。也会检查alpha值(透明度值)和物体之间的混合(Blend)。

图形渲染管线非常复杂，然而，对于大多数场合，我们必须做的只是顶点和片段着色器（因为GPU中没有默认的顶点/片段着色器）。几何着色器和细分着色器是可选的，通常使用默认的着色器就行了。

##顶点输入

####标准化设备坐标(Normalized Device Coordinates，NDC)：
开始绘制一些东西之前，我们必须给OpenGL输入一些顶点数据。只有当3个轴(x、y和z)在特定的-1.0到1.0的范围内时OpenGL才处理。所有在这个范围内的坐标叫做标准化设备坐标(Normalized Device Coordinates，NDC)，会最终显示在你的屏幕上(所有出了这个范围的都不会显示)。

如果我们要渲染一个2D三角形，它的顶点可以如此定义：

    GLfloat vertices[] = {
        -0.5f, -0.5f, 0.0f,
        0.5f, -0.5f, 0.0f,
        0.0f,  0.5f, 0.0f
    };	  

![]({{ site.url }}/images/post/2015-10-14/OpenGL Pipline 2.png)

NDC接着会变换为屏幕空间坐标(Screen-space Coordinates)，这是通过glViewport函数提供的数据，进行视口变换(Viewport Transform)完成的。最后的屏幕空间坐标被变换为像素输入到片段着色器。

####顶点缓冲对象(Vertex Buffer Objects, VBO)

#####VBO概念：
VBO为顶点缓冲区对象，用于存储顶点坐标/顶点uv/顶点法线/顶点颜色等数据信息。

比如前面我们有了顶点数据，就可以把它们作为输入数据传递给顶点着色器。通过VBO我们可以把需要渲染的图元的顶点信息，直接上传存储在GPU的显存中。使用这些缓冲对象的好处是可以一次性发送大批数据到显卡上，因为从CPU把数据发送到显卡相对较慢。当数据到了显卡内存中时，顶点着色器几乎立即就能获得顶点，这非常快。
VBO归根到底就是显卡存储空间里的一块缓存区(Buffer)而已，用于存储和顶点以及其属性相关的信息。这个Buffer有它的名字(VBO的ID)，OpenGL在GPU的某处记录着这个ID和对应的显存地址（或者地址偏移，类似内存）。

#####VBO的创建、配置:
生成一个缓冲ID：   
    
    GLuint VBO;
    glGenBuffers(1, &VBO);    

绑定新创建的缓冲：

    glBindBuffer(GL_ARRAY_BUFFER, VBO);  	

OpenGL有很多缓冲对象类型，GL_ARRAY_BUFFER是其中一个顶点缓冲对象的类型。上面我们把缓冲绑定到了GL_ARRAY_BUFFER类型上，OpenGL允许同时绑定多个缓冲，只要它们类型不同。
绑定之后，我们使用的任何(GL_ARRAY_BUFFER目标上的)缓冲函数都会用来配置当前绑定的缓冲(VBO)。
    
接下来调用glBufferData函数：
    
    glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);

它把用户定的义数据复制到当前绑定缓冲的内存中。    
现在，顶点数据发送给了GPU，但还没结束，OpenGL还不知道如何解释内存中的顶点数据，以及怎样把顶点数据链接到顶点着色器的属性上。我们需要告诉OpenGL怎么做。

#####链接顶点属性：
  
顶点着色器允许我们以任何想要的形式作为顶点属性(Vertex Attribute)的输入，它具有很强的灵活性，这意味着我们必须手动指定输入数据与顶点着色器顶点属性的对应关系。即必须在渲染前指定OpenGL如何解释顶点数据。	

我们的顶点缓冲数据被格式化为下面的形式：	
![]({{ site.url }}/images/post/2015-10-14/OpenGL Pipline 3.png)	

每个数据4个字节（32位），每个位置3个值，每个位置间没有间隙。

有这些信息我们就可以告诉OpenGL如何解释顶点数据了(每一个顶点属性)，使用glVertexAttribPointer这个函数：
    
    {% raw %}
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(GLfloat), (GLvoid*)0);
    glEnableVertexAttribArray(0);    
    {% endraw %}
        
glVertexAttribPointer的参数解释：

- 第一个指定顶点属性位置，与顶点着色器中 layout(location = 0) 对应。
- 第二个指定顶点属性大小。
- 第三个指定数据类型。
- 第四个定义是否希望数据被标准化。
- 第五个参数叫做步长(Stride)，指定在连续的顶点属性之间间隔有多少。由于我们下个位置数据在3个GLfloat之后，所以设为3 * sizeof(GLfloat)。
- 最后一个表示我们的位置数据在缓冲中起始位置的偏移量。

因为顶点属性默认是关闭的，所以之后要开启顶点属性，使用glEnableVertexAttribArray，把顶点属性位置值作为它的参数。
        
#####综合：
  
最终绘制一个物体，看起来会像这样： 

{% highlight C++ %}

// 0. 复制顶点数组到缓冲中提供给OpenGL使用
glBindBuffer(GL_ARRAY_BUFFER, VBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
// 1. 设置顶点属性指针
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(GLfloat), (GLvoid*)0);
glEnableVertexAttribArray(0);  
// 2. 当我们打算渲染一个物体时要使用着色器程序
glUseProgram(shaderProgram);
// 3. 绘制物体
someOpenGLFunctionThatDrawsOurTriangle();  
        
{% endhighlight %}
    
我们绘制一个物体的时候必须重复这件事。数据少时还好，但要绘制的顶点和物体很多时，绑定合适的缓冲对象，为每个物体配置所有顶点属性很快就变成一件麻烦事。有没有一些方法可以使我们把所有的配置储存在一个对象中，并且可以通过绑定这个对象来恢复状态？

####顶点数组对象(Vertex Array Object, VAO)

#####概念：

VAO就是所有顶点数据的状态集合。它存储了顶点数据的格式以及顶点数据所需的缓存对象的引用。    VAO一样可以绑定，任何随后的顶点属性调用都会储存在这个VAO中。这样的好处是，当配置顶点属性指针时，你只用做一次，每次绘制一个物体的时候，我们绑定相应VAO就行了。

记住：VAO中并没有存储顶点的相关属性数据。我们把顶点数据存储在数组中，然后放进VBO，最后在VAO中存储相关的状态。它的定位是state-object（状态对象，记录存储状态信息）。与buffer-object明显不同。
	
#####VBO与VAO：

综上所述，VAO相当于保存了顶点相关的各种信息，记录了各种状态。当我们要修改它的状态时，我们先激活它（glBindVertexArray()）。当我们要使用其中的状态时，如要绘图，也先激活它，然后调用相关的函数。

VBO在渲染阶段才指定数据位置和顶点信息，然后根据此信息去解析缓存区里的数据，联系这两者中间的桥梁是GL-Contenxt。GL-context整个程序一般只有一个，所以如果一个渲染流程里有两份不同的绘制代码，GL-context就负责在它们之间进行状态切换。这也是为什么要在渲染过程中，在每份绘制代码之中有glBindBuffer/glEnableVertexAttribArray/glVertexAttribPointer。	

![]({{ site.url }}/images/post/2015-10-14/OpenGL Pipline 4.png)
		
#####VAO生成、配置：

生成VAO与VBO类似：

    GLuint VAO;
    glGenVertexArrays(1, &VAO);

使用VAO要做的全部就是使用glBindVertexArray绑定VAO。自此我们就应该绑定/配置相应的VBO和属性指针。当我们打算绘制一个物体的时候，我们只要在绘制物体前简单地把VAO绑定到希望用到的配置就行了。

引用[这个博客](http://blog.csdn.net/candycat1992/article/details/39676669)的内容解释一下：
这个过程就像一个中介人的作用，而中介人就是GL_ARRAY_BUFFER​。我们可以这么想，glBindBuffer​ 设置了一个全局变量，然后glVertexAttribPointer读取了这个全局变量并把它存储在VAO中，这个全局变量就是GL_ARRAY_BUFFER。当调用完glVertexAttribPointer后，顶点属性已经知道了数据来源就是VBO，它们之间就会直接联系，而不需要在通过GL_ARRAY_BUFFER。
		
#####绘制函数glDrawArrays:

glDrawArrays函数为我们提供了绘制物体的能力，它使用当前激活的着色器、前面定义的顶点属性配置和VBO的顶点数据(通过VAO间接绑定)来绘制基本图形。
		
####索引缓冲对象(Element Buffer Objects，EBO):

索引缓冲对象简称EBO(或IBO)。举个例子：假设我们不再绘制一个三角形而是矩形。我们就可以绘制两个三角形来组成一个矩形(OpenGL主要就是绘制三角形)。这样我们就需要六个顶点集合，并且会有两个重合的顶点（左下角和右上角的点）。这样当顶点变多时，会产生很大的浪费。
所以最好的解决办法是每个顶点只存一次，当我们需要使用这些顶点时，只调用顶点的索引。这样我们只定义4个顶点就好。
        
索引缓冲的工作方式正是这样的。一个EBO是一个像顶点缓冲对象(VBO)一样的缓冲，它专门储存索引，OpenGL调用这些顶点的索引来绘制。我们先定义所有用到的独一无二的点，然后是绘制矩形的索引：

    GLfloat vertices[] = {
        0.5f, 0.5f, 0.0f,   // 右上角
        0.5f, -0.5f, 0.0f,  // 右下角
        -0.5f, -0.5f, 0.0f, // 左下角
        -0.5f, 0.5f, 0.0f   // 左上角
    };
        
    GLuint indices[] = { 
            // 起始于0!
        0, 1, 3, // 第一个三角形
        1, 2, 3  // 第二个三角形
    };


下一步我们需要创建索引缓冲对象：

    GLuint EBO;
    glGenBuffers(1, &EBO);

与VBO相似，我们绑定EBO然后用glBufferData把索引复制到缓冲里。缓冲的类型定义为GL_ELEMENT_ARRAY_BUFFER。而且和VBO相似，我们把这些函数调用放在绑定和解绑函数调用之间。
     
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW); 

要注意的是，我们现在用GL_ELEMENT_ARRAY_BUFFER当作缓冲目标。最后一件要做的事是用glDrawElements来替换glDrawArrays函数，来指明我们从索引缓冲渲染。当时用glDrawElements的时候，我们就会用当前绑定的索引缓冲进行绘制：    

    glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);


第一个参数指定了我们绘制的模式。第二个参数是我们打算绘制顶点的次数。第三个参数是索引的类型。最后一个参数里我们可以指定EBO中的偏移量。

glDrawElements函数从当前绑定到GL_ELEMENT_ARRAY_BUFFER目标的EBO获取索引。这意味着我们必须在每次要用索引渲染一个物体时绑定相应的EBO，这还是有点麻烦。不过顶点数组对象仍可以保存索引缓冲对象的绑定状态。VAO绑定之后可以索引缓冲对象，EBO就成为了VAO的索引缓冲对象。再次绑定VAO的同时也会自动绑定EBO。        
        
最终绘制三角形的代码请参考：[这里](http://learnopengl-cn.readthedocs.org/zh/latest/01%20Getting%20started/04%20Hello%20Triangle/)
        
---
> 参考：
[LearnOpenGL-CN : 你好，三角形](http://learnopengl-cn.readthedocs.org/zh/latest/01%20Getting%20started/04%20Hello%20Triangle/)