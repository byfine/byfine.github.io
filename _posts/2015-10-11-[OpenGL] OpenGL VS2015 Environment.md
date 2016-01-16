---
layout: post
title: OpenGL VS2015环境配置
description: "OpenGL环境配置简单教程"
modified: 2015-10-11
tags: [OpenGL]
---

### FreeGLUT 和 Glew

1. [FreeGLUT](http://freeglut.sourceforge.net/): 第三方库，可以用来显示窗口，管理用户输入，以及执行一些其他操作。
2. [GLEW](http://glew.sourceforge.net/)：跨平台第三方库，可以简化获取函数地址的过程，并且包含了可以跨平台使用的一些其他OpenGL编程方法。

有两种设置FreeGLUT和GLEW的方法：

1. 添加FreeGLUT和GLEW的库文件到VS的目录和系统目录，然后在VS配置，最后使用。
2. 添加FreeGLUT和GLEW的库文件到我们项目下自己建的一个目录，然后在VS中配置项目。这样当你的项目拷贝到其他没有FreeGLUT和GLEW的电脑，也可以运行。

我们使用第二种方法。



### 开始设置

1. 准备资源：   
	从[GLEW1.13.0](http://sourceforge.net/projects/glew/files/glew/1.13.0/glew-1.13.0-win32.zip/download)下载GLEW，并且解压出glew-1.13.0目录。

	从FreeGLUT官网下载3.0.0版本。但是FreeGLUT并没有编译，所以需要自己编译，这个过程比较麻烦需要CMAKE，所以我直接从这里下的[编译后的FreeGLUT](http://www.transmissionzero.co.uk/software/freeglut-devel/)，选for MSVC，下载后解压。
   
2. 新建一个VS项目：
	   
	打开VS2015，新建一个项目。选择Visual C++ 和 空项目。名字自己起，目录中不要有空格。

	然后在项目中新建一个 **main.cpp**文件。
   
3. 添加GLEW：
   
	在项目目录下，新建一个文件夹，取名**Dependencies**（当然你也可以取别的名字），在Dependencies下再建一个目录**glew**。

	到之前解压出的glew-1.13.0目录下，有一个include\GL目录，里面有三个.h文件，把这三个文件拷贝到Dependencies\glew目录下。

	在到glew-1.13.0\lib\Release目录，因为我是64位系统，所以选择x64目录下的glew32.lib拷贝到Dependencies\glew目录下。

	最后glew目录是这样：

	![glew目录]({{ site.url }}/images/post/2015-10-11/OpenGL+VS2015_1.png)
	   
4. 添加FreeGLUT：
   
	新建一个**freeglut**文件夹在**Dependencies**下。

	到之前下载解压出的freeglut目录下，include\GL内，有4个.h文件，将它们拷贝到Dependencies\freeglut。

	到之前下载解压出的freeglut目录下，lib\x64内，有一个freeglut.lib文件，同样拷贝到Dependencies\freeglut。

	最后freeglut目录是这样：

	![freeglut目录]({{ site.url }}/images/post/2015-10-11/OpenGL+VS2015_2.png)
   
5. 配置VS项目：

	回到VS2015，在 *解决方案资源管理器* 选中我们的项目，点击菜单**项目-显示所有文件**，再刷新一下 *解决方案资源管理器* ，会看到Dependencies出现，右键点击**包括在项目中**，再分别打开下面的目录，看看前面有红色图标的项目，也分别点**包括在项目中**。

	再选中项目，右键属性，打开属性窗口，选择**链接器-常规**，在**附加库目录**，输入 Dependencies\freeglut;Dependencies\glew 

	![链接器-输入]({{ site.url }}/images/post/2015-10-11/OpenGL+VS2015_3.png)

	再选择**链接器-输入**，在附加依赖项，加上 opengl32.lib;freeglut.lib;glew32.lib; 

	![链接器-输入]({{ site.url }}/images/post/2015-10-11/OpenGL+VS2015_4.png)

	点确定。 这时候就全部配置完了。

6. 测试：   
	main.cpp输入如下代码:

	{% highlight c++ %}
	# include "Dependencies\glew\glew.h"
	# include "Dependencies\freeglut\freeglut.h"

	void myDisplay(void)   
	{   
	   glClear(GL_COLOR_BUFFER_BIT);
	   glColor3f(0.0f, 1.0f, 0.0f);
	   glRectf(-0.5f, -0.5f, 0.5f, 0.5f);
	   glFlush();   
	}

	int main(int argc, char *argv[])   
	{
	   glutInit(&argc, argv);
	   glutInitDisplayMode(GLUT_RGB | GLUT_SINGLE);
	   glutInitWindowPosition(100, 100);
	   glutInitWindowSize(640, 480);
	   glutCreateWindow("First_GL!");
	   glutDisplayFunc(myDisplay);
	   glutMainLoop();
	}   
	{% endhighlight %}

### 注意：

F5运行，如果弹出提示找不到freeglut.dll，回到下载的freeglut\bin\x64目录，把freeglut.dll拷贝到VS项目的Debug目录（和.sln文件目录同级，x64\debug）即可。

还有对于64位系统，不要忘记在VS把平台改成x64。

