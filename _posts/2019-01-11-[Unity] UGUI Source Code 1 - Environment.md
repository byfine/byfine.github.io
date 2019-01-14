---
layout: post
title: UGUI源码分析 1 - 环境搭建 
description: "UGUI源码，环境搭建"
modified: 2019-01-11
tags: [Unity]
---

Unity 已经将UGUI源码发布了很久，作为经常使用UGUI的程序员，以前一直对其一知半解，现在有点空闲，因此决定把源码学习一下。

## 环境搭建：

1. 下载源码文件  
    在[Unity Bitbucket](https://bitbucket.org/Unity-Technologies/ui/downloads/?tab=tags)上下载对应版本的压缩包。然后解压。  
    我的环境是win10, unity 2018.3f2。

2. 编译代码
    解压完，其实按照readme的描述，就可以用vs打开编译，然后把编译出来的dll文件，替换安装目录  `Data\UnityExtensions\Unity\GUISystem\{UNITY_VERSION}` 下的文件。
    你可以在源码文件里加一句log代码，测试是否成功，比如 UnityEngine.UI\UI\Button.cs 的 press 函数加一句log，在项目里新建一个button，点击就会打印log信息。  
    但是这个方法调试起来非常麻烦，虽然可以用pdb转成mdb文件，让mono进行调试，但是每次改动都要重新生成，所以比较好的方法是创建一个可以直接修改编译UI代码的项目。

3. 创建测试项目
    1. 打开Unity安装目录 `Editor\Data\UnityExtensions\Unity` 可以发现里面有很多Unity功能的文件夹，这些对Unity来说其实相当于外部插件，删除也不会影响Unity运行，只是无法使用相关功能。所以我们先把GUISystem备份一下，然后将其删除。  
    2. 接着新建一个Unity项目。  
    3. 把UGUI源码里的cs代码拷贝到项目里：可以发现只有 UnityEditor.UI 和 UnityEngine.UI 两个文件夹下有cs文件，一个是editor相关代码，一个是源码。把 Properties 和 .csproj 等不相关文件删除，然后把两个文件夹拷贝到项目内，并把 UnityEditor.UI 文件重命名为 Editor，因为这样Unity才能识别。
        ![]({{ site.url }}/images/post/UGUI-Source/Folders.jpg)
    4. 这时候编译项目，会发现一大堆报错。不用担心，看一下会发现都是一些 package功能的报错，因为我们只是测试源码的项目，不需要这些功能， 直接打开 Package Manager， 把不相关的插件都remove，关掉项目重新打开，编译会发现可以通过了。（如果项目没有编译通过，应该是打不开 Package Manager 的，这时候可以打开项目目录 Packages\manifest.json 文件，手动删除对应的插件即可）。
    5. 此时就可以随便修改UGUI源码，并很容易进行调试，方便我们阅读源码了~~


