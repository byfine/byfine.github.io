---
layout: post
title: OpenGL 第一个窗口程序
description: "OpenGL 第一个窗口程序"
modified: 2015-10-13
tags: [OpenGL]
---

我们参考 LearnOpenGL-CN，使用GLFW和GLEW库来创建第一个窗口程序。
 
要画出各种效果，首先需要一个OpenGL上下文(Context)和一个用于显示的窗口。我们知道，OpenGL是跨平台的，这些操作在每个系统上都是不一样的，所以OpenGL有目的的抽象(Abstract)这些操作。它并不会实现这些与硬件和系统相关的具体操作。所以我们需要依靠第三方库。
 
常见的第三方库GLUT，SDL，SFML，GLFW，GLEW等等。之前也介绍过了FreeGLUT和GLEW的安装方法。各个第三方库的使用方法都差不多，从它们官网上下头文件、Lib文件等等，放项目里设置好依赖关系。具体的可以参考[这篇教程](http://learnopengl-cn.readthedocs.org/zh/latest/01%20Getting%20started/02%20Creating%20a%20window/)。
 
### GLFW
GLFW是一个专门针对OpenGL的C语言库，它提供了一些渲染物件所需的最低限度的接口。它允许用户创建OpenGL上下文，定义窗口参数以及处理用户输入。
 
### GLEW
因为OpenGL只是一个规范，具体的实现是由驱动开发商针对特定显卡实现的。由于显卡驱动版本众多，大多数函数都无法在编译时确定下来，需要在运行时获取。开发者需要运行时获取函数地址并保存下来供以后使用。Windows下类似这样：

{% highlight C++ %}
// 定义函数类型
typedef void (*GL_GENBUFFERS) (GLsizei, GLuint*);
// 找到正确的函数并赋值给函数指针
GL_GENBUFFERS glGenBuffers  = (GL_GENBUFFERS)wglGetProcAddress("glGenBuffers");
// 现在函数可以被正常调用了
GLuint buffer;
glGenBuffers(1, &buffer);
{% endhighlight %}

对于每个函数都必须这样，简直无情。幸运的是，有一个针对此目的的库，GLEW，是目前最流行的做这件事的方式。
 
### 创建窗口
配置好GLFW和GLEW后，就可以开始创建我们的第一个程序了。VS新建项目啥的就不说了。
下面只是大概介绍，具体内容可以参考[这里](http://learnopengl-cn.readthedocs.org/zh/latest/01%20Getting%20started/03%20Hello%20Window/)。
 
#### 头文件
先包含头文件。我们使用GLEW的静态链接库，所以定义了宏GLEW_STATIC。
（静态链接库就是.lib文件，编译时会把库中的代码合并到可执行文件里。动态链接库是指一个库通过.dll或.so的方式存在，它的代码与你的可执行文件是分离的。发布时必须带上这些dll。）

{% highlight C++ %}
#define GLEW_STATIC  
#include <GL/glew.h>
#include <GLFW/glfw3.h>

//窗口大小
const GLuint WIDTH = 800, HEIGHT = 600;
//窗口对象
GLFWwindow* window;
{% endhighlight %}

#### 初始化GLFW
  接下来我们定义一个函数，执行初始化GLFW的操作：
  
{% highlight C++ %}
int Init_GLFW()
{
    // Init GLFW
    glfwInit();
    // Set all the required options for GLFW
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3); //主版本号
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3); //副版本号
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE); //核心模式
    glfwWindowHint(GLFW_RESIZABLE, GL_FALSE); //窗口不可调大小

    // Create a GLFWwindow object that we can use for GLFW's functions
    window = glfwCreateWindow(WIDTH, HEIGHT, "LearnOpenGL", nullptr, nullptr);
    //检测是否创建成功
    if (window == nullptr)
    {
        std::cout << "Failed to create GLFW window" << std::endl;
        glfwTerminate();
        return -1;
    }
    
    glfwMakeContextCurrent(window);

    // 注册键盘回掉函数
    glfwSetKeyCallback(window, key_callback);

    return 0;
}
{% endhighlight %}

先调glfwInit初始化，然后一堆glfwWindowHint函数进行配置。然后创建了一个窗口对象GLFWwindow，这个窗口对象中具有和窗口相关的许多数据，而且会被GLFW的其他函数频繁地用到。glfwMakeContextCurrent通知GLFW给我们的窗口在当前的线程中创建我们等待已久的OpenGL上下文。
 
#### 初始化GLEW
再来个函数，初始化GLEW：

{% highlight C++ %}
int Init_GLEW()
{
    glewExperimental = GL_TRUE;
    // Initialize GLEW to setup the OpenGL Function pointers
    if (glewInit() != GLEW_OK)
    {
        std::cout << "Failed to initialize GLEW" << std::endl;
        return -1;
    }

    // 设置窗口大小
    glViewport(0, 0, WIDTH, HEIGHT);

    return 0;
}
{% endhighlight %}

glewExperimental设为True，能让GLEW在管理OpenGL的函数指针时更多地使用现代化的技术。设为GL_FALSE的话可能会在使用OpenGL的核心模式(Core-profile)时出现一些问题。

OpenGL使用glViewport定义的位置和宽高进行位置坐标的转换，将OpenGL中的位置坐标转换为你的屏幕坐标。例如，OpenGL中的坐标(0.5,0.5)有可能被转换为屏幕中的坐标(200,450)。注意，OpenGL只会把-1到1之间的坐标转换为屏幕坐标，因此在此例中(-1，1)转换为屏幕坐标是(0,600)。

main函数里先调用这俩函数。

#### 循环
我们不希望窗口只绘制一个图像后就关闭。可以在程序中添加一个while循环，这样程序就能在我们让GLFW退出前保持运行了。

{% highlight C++ %}
while(!glfwWindowShouldClose(window))
{
    glfwPollEvents();
    glfwSwapBuffers(window);
}
{% endhighlight %}

glfwWindowShouldClose，检查GLFW是否准备好要退出。
glfwPollEvents， 检查有没有触发什么事件(比如键盘有按钮按下、鼠标移动等)然后调用对应的回调函数。
glfwSwapBuffers，会交换缓冲区

最后，当游戏循环结束后我们需要释放之前的操作分配的资源，在main函数的最后加入如下代码：

{% highlight C++ %}
glfwTerminate();
return 0;
{% endhighlight %}

这时候运行就会有个窗口了。。。没有的话去看前面介绍的原网站。。。
 
#### 输入
GLFW中的键盘、鼠标等控制是通过回调函数，就是函数指针，我们注册好后，在事件触发后会调用该函数。跟C#委托和事件差不多。前面GLFW初始化时我们已经过一个键盘回调函数，它具体如下：

{% highlight C++ %}
glfwSetKeyCallback(window, key_callback);  //注册函数
...
...

void key_callback(GLFWwindow* window, int key, int scancode, int action, int mode)
{
    // 当用户按下ESC键,我们设置window窗口的WindowShouldClose属性为true
    // 关闭应用程序
    if(key == GLFW_KEY_ESCAPE && action == GLFW_PRESS){
        glfwSetWindowShouldClose(window, GL_TRUE);
    }
}    
{% endhighlight %}

#### 渲染
我们要把所有的渲染操作放到循环中，因为我们想让这些渲染操作在每次循环迭代的时候都能被执行。
为了测试，我们让屏幕清空为一种颜色。可以通过调用glClear函数来清空屏幕缓冲区的颜色.

{% highlight C++ %}
// 程序循环
while(!glfwWindowShouldClose(window))
{
    // 检查事件
    glfwPollEvents();
 
    // 在这里执行各种渲染操作
    ...
    glClearColor(0.2f, 0.3f, 0.3f, 1.0f);
    glClear(GL_COLOR_BUFFER_BIT);
 
    //交换缓冲区
    glfwSwapBuffers(window);
}
{% endhighlight %}

---
> 参考：
[你好，窗口](http://learnopengl-cn.readthedocs.org/zh/latest/01%20Getting%20started/03%20Hello%20Window/)