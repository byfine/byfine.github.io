---
layout: post
title: OpenGL 几何着色器(Geometry Shader)
description: "OpenGL几何着色器(Geometry Shader)简介"
modified: 2015-10-29
tags: [OpenGL]
---

##几何着色器(Geometry Shader)

在顶点和片段着色器之间有一个可选的着色器，叫做几何着色器（geometry shader）。几何着色器以一个或多个表示为一个基本图元（primitive）的顶点作为输入，比如可以是一个点或者三角形。几何着色器在将这些顶点发送到下一个着色阶段之前，可以将这些顶点转变为它认为合适的内容。几何着色器有意思的地方在于它可以把（一个或多个）顶点转变为完全不同的基本图形（primitive），从而生成比原来多得多的顶点。

我们用一个例子了解一下：

    #version 330 core
    layout (points) in;
    layout (line_strip, max_vertices = 2) out;

    void main() {
        gl_Position = gl_in[0].gl_Position + vec4(-0.1, 0.0, 0.0, 0.0);
        EmitVertex();

        gl_Position = gl_in[0].gl_Position + vec4(0.1, 0.0, 0.0, 0.0);
        EmitVertex();

        EndPrimitive();
    }

每个几何着色器开始位置我们需要声明输入的基本图形(primitive)类型，这个输入是我们从顶点着色器中接收到的。我们在in关键字前面声明一个layout标识符。这个输入layout修饰符可以从一个顶点着色器接收以下基本图形值：

|基本图形|	描述|
|:---------: |:--------: |
|points	|绘制GL_POINTS基本图形的时候（1）|
|lines|	当绘制GL_LINES或GL_LINE_STRIP（2）时|
|lines_adjacency|	GL_LINES_ADJACENCY或GL_LINE_STRIP_ADJACENCY（4）|
|triangles	|GL_TRIANGLES, GL_TRIANGLE_STRIP或GL_TRIANGLE_FAN（3）|
|triangles_adjacency|	GL_TRIANGLES_ADJACENCY或GL_TRIANGLE_STRIP_ADJACENCY（6）|

这是我们能够给渲染函数的几乎所有的基本图形。如果我们选择以GL_TRIANGLES绘制顶点，我们要把输入修饰符设置为triangles。括号里的数字代表一个基本图形所能包含的最少的顶点数。

当我们需要指定一个几何着色器所输出的基本图形类型时，我们就在out关键字前面加一个layout修饰符。和输入layout标识符一样，输出的layout标识符也可以接受以下基本图形值：

* points
* line_strip
* triangle_strip

使用这3个输出修饰符我们可以从输入的基本图形创建任何我们想要的形状。为了生成一个三角形，我们定义一个triangle_strip作为输出，然后输出3个顶点。

几何着色器同时希望我们设置一个它能输出的顶点数量的最大值（如果你超出了这个数值，OpenGL就会忽略剩下的顶点），我们可以在out关键字的layout标识符上做这件事。在这个特殊的情况中，我们将使用最大值为2个顶点，来输出一个line_strip。

这种情况，你会奇怪什么是线条：一个线条是把多个点链接起来表示出一个连续的线，它最少有两个点来组成。每多一个点都会在这个新点和之前一个点渲染线，如下图，其中包含5个顶点：

![]({{ site.url }}/images/post/2015-10-29/OpenGL-Geometry-1.png)

上面的着色器，我们只能输出一个线段，因为顶点的最大值设置为2。

为生成更有意义的结果，我们需要某种方式从前一个着色阶段获得输出。GLSL为我们提供了一个内建变量，它叫做gl_in，它的内部看起来可能像这样：

    in gl_Vertex
    {
        vec4 gl_Position;
        float gl_PointSize;
        float gl_ClipDistance[];
    } gl_in[];

这里它被声明为一个接口块(interface block)，它包含几个有意思的变量，其中最有意思的是gl_Position，它包含着和我们设置的顶点着色器的输出相似的向量。

要注意的是，它被声明为一个数组，因为大多数渲染基本图形由一个以上顶点组成，几何着色器接收一个基本图形的所有顶点作为它的输入。

使用来自前一个顶点着色阶段的顶点数据，我们就可以开始生成新的数据了，这是通过2个几何着色器函数EmitVertex和EndPrimitive来完成的。几何着色器需要你去生成/输出至少一个你定义为输出的基本图形。在我们的例子里我们打算至少生成一个线条（line strip）基本图形。

每次我们调用EmitVertex，当前设置到gl_Position的向量就会被添加到基本图形上。无论何时调用EndPrimitive，所有为这个基本图形发射出去的顶点都将结合为一个特定的输出渲染基本图形。一个或多个EmitVertex函数调用后，重复调用EndPrimitive就能生成多个基本图形。这个特殊的例子里，发射了两个顶点，它们被从顶点原来的位置平移了一段距离，然后调用EndPrimitive将这两个顶点结合为一个单独的有两个顶点的线条。

现在你了解了几何着色器的工作方式，你就可能猜出这个几何着色器做了什么。这个几何着色器接收一个基本图形——点，作为它的输入，使用输入点作为它的中心，创建了一个水平线基本图形。如果我们渲染它，结果就会像这样：

![]({{ site.url }}/images/post/2015-10-29/OpenGL-Geometry-2.png)

并不是非常引人注目，但是考虑到它的输出是使用下面的渲染命令生成的就很有意思了：

    glDrawArrays(GL_POINTS, 0, 4);

这是个相对简单的例子，它向你展示了我们如何使用几何着色器来动态地在运行时生成新的形状。


##使用几何着色器

为了展示几何着色器的使用，我们将渲染一个简单的场景，在场景中我们只绘制4个点，这4个点在标准化设备坐标的z平面上。这些点的坐标是：

    GLfloat points[] = {
    -0.5f,  0.5f, // 左上方
    0.5f,  0.5f,  // 右上方
    0.5f, -0.5f,  // 右下方
    -0.5f, -0.5f  // 左下方
    };

顶点着色器只在z平面绘制点，所以我们只需要一个基本顶点着色器：

    #version 330 core
    layout (location = 0) in vec2 position;

    void main()
    {
        gl_Position = vec4(position.x, position.y, 0.0f, 1.0f);
    }

我们会简单地为所有点输出绿色，我们直接在片段着色器里进行硬编码：

    #version 330 core
    out vec4 color;

    void main()
    {
        color = vec4(0.0f, 1.0f, 0.0f, 1.0f);
    }


为点的顶点生成一个VAO和VBO，然后使用glDrawArrays进行绘制：

    shader.Use();
    glBindVertexArray(VAO);
    glDrawArrays(GL_POINTS, 0, 4);
    glBindVertexArray(0);

效果是黑色场景中有四个绿点。现在我们将为场景添加一个几何着色器。

出于学习的目的我们将创建一个叫pass-through的几何着色器，它用一个point基本图形作为它的输入，并把它无修改地传（pass）到下一个着色器。

    #version 330 core
    layout (points) in;
    layout (points, max_vertices = 1) out;
    void main() {
        gl_Position = gl_in[0].gl_Position;
        EmitVertex();
        EndPrimitive();
    }

现在这个几何着色器应该很容易理解了。它简单地将它接收到的输入的无修改的顶点位置发射出去，然后生成一个point基本图形。

一个几何着色器需要像顶点和片段着色器一样被编译和链接，但是这次我们将使用GL_GEOMETRY_SHADER作为着色器的类型来创建这个着色器：

    geometryShader = glCreateShader(GL_GEOMETRY_SHADER);
    glShaderSource(geometryShader, 1, &gShaderCode, NULL);
    glCompileShader(geometryShader);  
    ...
    glAttachShader(program, geometryShader);
    glLinkProgram(program);

编译着色器的代码和顶点、片段着色器的基本一样。要记得检查编译和链接错误！
运行后效果和前面一样，还是四个绿点。


##创建几个房子

绘制点和线没什么意思，所以我们将在每个点上使用几何着色器绘制一个房子。我们可以通过把几何着色器的输出设置为triangle_strip来达到这个目的，总共要绘制3个三角形：两个用来组成方形和另表示一个屋顶。

在OpenGL中三角形带(triangle strip)绘制起来更高效，因为它所使用的顶点更少。第一个三角形绘制完以后，每个后续的顶点会生成一个毗连前一个三角形的新三角形：每3个毗连的顶点都能构成一个三角形。如果我们有6个顶点，它们以三角形带的方式组合起来，那么我们会得到这些三角形：（1, 2, 3）、（2, 3, 4）、（3, 4, 5）、（4,5,6）因此总共可以表示出4个三角形。一个三角形带至少要用3个顶点才行，它能生成N-2个三角形；6个顶点我们就能创建6-2=4个三角形。下面的图片表达了这点：

![]({{ site.url }}/images/post/2015-10-29/OpenGL-Geometry-3.png)

使用一个三角形带作为一个几何着色器的输出，我们可以轻松创建房子的形状，只要以正确的顺序来生成3个毗连的三角形。下面的图像显示，我们需要以何种顺序来绘制点，才能获得我们需要的三角形，图上的蓝点代表输入点：

![]({{ site.url }}/images/post/2015-10-29/OpenGL-Geometry-4.png)

上图的内容转变为几何着色器：

    #version 330 core
    layout (points) in;
    layout (triangle_strip, max_vertices = 5) out;

    void build_house(vec4 position)
    {
        gl_Position = position + vec4(-0.2f, -0.2f, 0.0f, 0.0f);// 1:左下角
        EmitVertex();
        gl_Position = position + vec4( 0.2f, -0.2f, 0.0f, 0.0f);// 2:右下角
        EmitVertex();
        gl_Position = position + vec4(-0.2f,  0.2f, 0.0f, 0.0f);// 3:左上
        EmitVertex();
        gl_Position = position + vec4( 0.2f,  0.2f, 0.0f, 0.0f);// 4:右上
        EmitVertex();
        gl_Position = position + vec4( 0.0f,  0.4f, 0.0f, 0.0f);// 5:屋顶
        EmitVertex();
        EndPrimitive();
    }

    void main()
    {
        build_house(gl_in[0].gl_Position);
    }

这个几何着色器生成5个顶点，每个顶点是点（point）的位置加上一个偏移量，来组成一个大三角形带。接着最后的基本图形被像素化，片段着色器处理整三角形带，结果是为我们绘制的每个点生成一个绿房子。
可以看到，每个房子实则是由3个三角形组成，都是仅仅使用空间中一点来绘制的。绿房子看起来还是不够漂亮，所以我们再给每个房子加一个不同的颜色。我们将在顶点着色器中为每个顶点增加一个额外的代表颜色信息的顶点属性。

下面是更新了的顶点数据：

    GLfloat points[] = {
        -0.5f,  0.5f, 1.0f, 0.0f, 0.0f, // 左上
        0.5f,  0.5f, 0.0f, 1.0f, 0.0f, // 右上
        0.5f, -0.5f, 0.0f, 0.0f, 1.0f, // 右下
        -0.5f, -0.5f, 1.0f, 1.0f, 0.0f  // 左下
    };
    
然后我们更新顶点着色器，使用一个接口块来向几何着色器发送颜色属性：

    #version 330 core
    layout (location = 0) in vec2 position;
    layout (location = 1) in vec3 color;

    out VS_OUT {
        vec3 color;
    } vs_out;

    void main()
    {
        gl_Position = vec4(position.x, position.y, 0.0f, 1.0f);
        vs_out.color = color;
    }

接着我们还需要在几何着色器中声明同样的接口块(使用一个不同的接口名)：

    in VS_OUT {
        vec3 color;
    } gs_in[];

因为几何着色器把多个顶点作为它的输入，从顶点着色器来的输入数据总是被以数组的形式表示出来，即使现在我们只有一个顶点。

然后我们还要为下一个像素着色阶段声明一个输出颜色向量：

    out vec3 fColor;

因为片段着色器只需要一个颜色，传送多个颜色没有意义。fColor向量这样就不是数组。当发射一个顶点时，为了它的片段着色器运行，每个顶点都会储存最后在fColor中储存的值。对于这些房子来说，我们可以在第一个顶点被发射，对整个房子上色前，只使用来自顶点着色器的颜色填充fColor一次：

    fColor = gs_in[0].color; //只有一个输出颜色，所以直接设置为gs_in[0]

使用几何着色器，你可以使用最简单的基本图形就能获得漂亮的新玩意。因为这些形状是在你的GPU超快硬件上动态生成的，这要比使用顶点缓冲自己定义这些形状更为高效。几何缓冲在简单的经常被重复的形状比如体素（voxel）的世界和室外的草地上，是一种非常强大的优化工具。

##爆炸式物体

当我们说对一个物体进行爆破的时候并不是说我们将要把之前的那堆顶点炸掉，而是把每个三角形沿着它们的法线向量移动一小段距离。效果是整个物体上的三角形看起来就像沿着它们的法线向量爆炸了一样。这样一个几何着色器效果的一大好处是，它可以用到任何物体上，无论它们多复杂。

因为我们打算沿着三角形的法线向量移动三角形的每个顶点，我们需要先计算它的法线向量。我们可以使用叉乘获取一个垂直于两个其他向量的向量。下面的几何着色器函数做的正是这件事，它使用3个输入顶点坐标获取法线向量：

    vec3 GetNormal()
    {
    vec3 a = vec3(gl_in[0].gl_Position) - vec3(gl_in[1].gl_Position);
    vec3 b = vec3(gl_in[2].gl_Position) - vec3(gl_in[1].gl_Position);
    return normalize(cross(a, b));
    }
    
一定要注意，如果我们调换了a和b的叉乘顺序，我们得到的法线向量就会使反的，顺序很重要！

接下来我们创建一个explode函数，函数返回的是一个新向量，它把位置向量沿着法线向量方向平移：

    vec4 explode(vec4 position, vec3 normal)
    {
        float magnitude = 2.0f;
        vec3 direction = normal * ((sin(time) + 1.0f) / 2.0f) * magnitude;
        return position + vec4(direction, 0.0f);
    }
    
sin函数把一个time变量作为它的参数，它根据时间来返回一个-1.0到1.0之间的值。因为我们不想让物体坍缩，所以我们把sin返回的值做成0到1的范围。最后的值去乘以法线向量，direction向量被添加到位置向量上。

爆炸效果的完整的几何着色器是这样的，它使用我们的模型加载器，绘制出一个模型：

    #version 330 core
    layout (triangles) in;
    layout (triangle_strip, max_vertices = 3) out;

    in VS_OUT {
        vec2 texCoords;
    } gs_in[];

    out vec2 TexCoords;

    uniform float time;

    vec4 explode(vec4 position, vec3 normal) { ... }

    vec3 GetNormal() { ... }

    void main() {
        vec3 normal = GetNormal();

        gl_Position = explode(gl_in[0].gl_Position, normal);
        TexCoords = gs_in[0].texCoords;
        EmitVertex();
        gl_Position = explode(gl_in[1].gl_Position, normal);
        TexCoords = gs_in[1].texCoords;
        EmitVertex();
        gl_Position = explode(gl_in[2].gl_Position, normal);
        TexCoords = gs_in[2].texCoords;
        EmitVertex();
        EndPrimitive();
    }
    
注意我们同样在发射一个顶点前输出了合适的纹理坐标。

最后的结果是一个随着时间持续不断地爆炸的3D模型（不断爆炸不断回到正常状态）。尽管没什么大用处，它却向你展示出很多几何着色器的高级用法。


##把法线向量显示出来

在这部分我们将使用几何着色器写一个例子，非常有用：显示一个法线向量。当编写光照着色器的时候，你最终会遇到奇怪的视频输出问题，你很难决定是什么导致了这个问题。通常导致光照错误的是，不正确的加载顶点数据，以及给它们指定了不合理的顶点属性，又或是在着色器中不合理的管理，导致产生了不正确的法线向量。我们所希望的是有某种方式可以检测出法线向量是否正确。把法线向量显示出来正是这样一种方法，恰好几何着色器能够完美地达成这个目的。

思路是这样的：我们先不用几何着色器，正常绘制场景，然后我们再次绘制一遍场景，但这次只显示我们通过几何着色器生成的法线向量。几何着色器把一个三角形基本图形作为输入类型，用它们生成3条和法线向量同向的线段，每个顶点一条。伪代码应该是这样的：

这次我们会创建一个使用模型提供的顶点法线，而不是自己去生成。为了适应缩放和旋转，我们会在变换到裁切空间坐标前，先使用正规矩阵来变换法线。（几何着色器用他的位置向量做为裁切空间坐标，所以我们还要把法线向量变换到同一个空间）。这些都能在顶点着色器中完成：

    #version 330 core
    layout (location = 0) in vec3 position;
    layout (location = 1) in vec3 normal;

    out VS_OUT {
        vec3 normal;
    } vs_out;

    uniform mat4 projection;
    uniform mat4 view;
    uniform mat4 model;

    void main()
    {
        gl_Position = projection * view * model * vec4(position, 1.0f); 
        mat3 normalMatrix = mat3(transpose(inverse(view * model)));
        vs_out.normal = normalize(vec3(projection * vec4(normalMatrix * normal, 1.0)));
    }

经过变换的裁切空间法线向量接着通过一个接口块被传递到下个着色阶段。几何着色器接收每个顶点（带有位置和法线向量），从每个位置向量绘制出一个法线向量：

    #version 330 core
    layout (triangles) in;
    layout (line_strip, max_vertices = 6) out;

    in VS_OUT {
        vec3 normal;
    } gs_in[];

    const float MAGNITUDE = 0.4f;

    void GenerateLine(int index)
    {
        gl_Position = gl_in[index].gl_Position;
        EmitVertex();
        gl_Position = gl_in[index].gl_Position + vec4(gs_in[index].normal, 0.0f) * MAGNITUDE;
        EmitVertex();
        EndPrimitive();
    }

    void main()
    {
        GenerateLine(0); // First vertex normal
        GenerateLine(1); // Second vertex normal
        GenerateLine(2); // Third vertex normal
    }  

由于把法线显示出来通常用于调试的目的，我们可以在片段着色器的帮助下把它们显示为单色的线（如果你愿意也可以更炫一点）。

    #version 330 core
    out vec4 color;

    void main()
    {
        color = vec4(1.0f, 1.0f, 0.0f, 1.0f);
    }


---
> 参考
[LearnOpenGL-CN 几何着色器](http://learnopengl-cn.readthedocs.org/zh/latest/04%20Advanced%20OpenGL/09%20Geometry%20Shader/)