---
layout: post
title:  Tools：Visual Studio Code 简介 及在Unity中使用 
description: "Visual Studio Code 简介 及在Unity中使用"
modified: 2015-11-11
tags: [Tools]
---

https://code.visualstudio.com/docs/editor/codebasics

VS Code 是微软推出的跨平台编辑器。它采用常见的UI布局，左侧是explorer 显示你打开的文件和文件夹，右侧是编辑窗口。
另外还有需要多额外的特性。

## Files, Folders & Projects
VS Code 是基于文件和文件夹的，你可以直接打开一个文件或者文件夹。另外，VSCode可以读取许多工具和平台定义的项目文件。如果你打开的文件夹有这些项目文件比如 project.json 或是 Visual Studio 的项目文件，它都会自动读取它们并提供更多功能，比如智能提示等。

## 基本布局
VS Code 的 UI 被划分为四个区域：

- Editor 是你编辑文件的地方，可以最多并排打开三个窗口。
- Side Bar 包含不同的视图窗口，比如Explorer。
- Status Bar 指示当前打开的文件和项目的信息。
- View Bar 在最左侧，让你切换不同视图，并有额外的上下文提示。

注：你可以把Side Bar移到右边（View->Move Sidebar），或者用Ctrl+B切换显示关闭。

## 并排编辑
你可以同时打开三个editor进行编辑。有多种方式打开一个新的editor：

- Ctrl并点击Explorer中的文件。
- Ctrl+\ 把当前editor分为两个。
- Open to the Side  来自Explorer窗口的命令。

当你有多个editor打开，可以使用Ctrl加数字1，2，3来切换不同窗口。

## Explorer
用来浏览、打开和管理项目所有文件和文件夹。
一旦打开一个文件夹，其内容就会显示在Explorer中。你可以对它们进行许多操作：

- 创建、删除、重命名文件和文件夹。
- 通过拖拽移动文件和文件夹。
- 用右键菜单查看更多操作。
	
你可把外部文件拖到VSCode来复制它们。
默认情况下，VSCode会排除一些文件夹（比如.git）。可以使用 files.exclude配置规则。
这对于排除自动生成的文件很有用，比如Unity自动生成的 *.meta文件。

注：使用Ctrl+P可以快速搜索并打开文件。

## Working Files
在Explorer 顶部有一个 Working Files 标签。这是激活文件的列表。一些情况下，文件会出现在这个列表：

- 修改一个文件。
- 双击打开一个文件。
- 打开一个不是当前文件夹的文件。

![]({{ site.url }}/images/post/VSCode/VSCode-1.png)

重新点击文件就会再次激活它们，不需要时关闭就即可。

注：你可在设置中配置 Working Files 的样式，比如explorer.workingFiles.maxVisible设置列表最大数量，explorer.workingFiles.dynamicHeight 设置是否自动设置高度。

## Save/Auto Save
默认情况，VSCode需要你手动保存文件，使用Ctrl+S。
当然，你也可以使用自动保存。
在命令板可以设置，按F1，然后输入auto设置。也可以在File菜单选项设置。

## 搜索文件
使用Ctrl+Shift+F可以快速搜索当前打开文件。
同时支持正则表达式搜索。

也可以使用Ctrl+Shift+J开启高级搜索。

![]({{ site.url }}/images/post/VSCode/VSCode-2.png)

通过Search下面两个输入栏，你可以指示包括或排除哪些文件。
点击左侧按钮，可以开启glob pattern syntax：

- \* 匹配一个或多个字符
- ? 匹配一个字符
- ** 匹配任何数量路径片段，包括空
- {}  群组条件 （比如  {**/*.html,**/*.txt}，搜索所有html和txt文件）
- []  指示字符范围（比如 example.[0-9] 匹配 example.0, example.1, …）

## Command Palette
F1可以打开命令面板。你可以获取所有VSCode的功能。
这个面板UI可以实现许多功能，下面是另外一些有用的快捷键：

- Ctrl+P 输入文件名打开文件
- Ctrl+Tab 循环之前打开的文件
- F1
- Ctrl+Shift+O 跳转到特定符合。
- Ctrl+G 跳转到特定行号。

输入?查看可以使用的指令。

## 快速文件导航
按住Ctrl 再按Tab 会显示一个所有打开过的文件列表，使用Tab切换文件，当选到某个要打开的文件，松开Ctrl就可以打开它。
另外，也可以使用Alt+Left 和 Alt+Right切换文件和编辑位置。

## 文件编码支持
在 User Settings 或 Workspace Settings 的 files.encoding，可以分别设置全局和当前工作空间的编码格式。
在状态栏可以查看当前编码格式，点击可以重新保存为新的格式。

## 命令行启动
可以使用命令行启动VSCode（前提是已经加入PATH），只需输入：

    Code .
    
也可以打开或者创建文件，使用空格分隔任意多个文件，文件存在会打开，不存在会创建新文件：

    code index.html style.css readme.md

## 设置
有两种设置：User和Workspace
User是全局的，Workspace只保存在当前工作空间，会重写User设置。

## 编辑器进阶

- Ctrl+Shift+] 跳转到匹配的括号。
- 选择 和 多光标:
  Alt+Click 可以多处选择，Ctrl+Alt+Down 或 Ctrl+Alt+Up可以插入多个光标。
  Ctrl+D 选择光标指的整个单词，Ctrl+K Ctrl+D 将最后添加的光标移向当前选择内容下一次出现的位置。
  Ctrl+Shift+L 会选择所有当前选择的内容，还有Ctrl+F2在各个内容切换。
- Shift+Alt+Left 和 Shift+Alt+Right，收缩和扩大选择范围。
- F12转到定义，Ctrl并停到一个位置，会显示预览。
- 在 C# 和 TypeScript中，可以使用Ctrl + T 跳转到任意符号。
- F2重命名。
- Ctrl+Shift+M显示所有错误。


### 连接Unity 和 VS Code
最简单的方法是利用Unity plug-in。

1. 下载插件

    git clone https://github.com/dotBunny/VSCode.git
  
2. 把plug-in添加进你的Unity项目
    把刚下载的文件添加进Unity。然后打开Unity Preferences，选择VSCode窗口：	
    ![]({{ site.url }}/images/post/VSCode/VSCode-3.png)

    勾选Enable Integration，就会开启功能。
    点击Write Workspace Settings，会进行相关配置，比如在VSCode排除无关文件（.meta）等。
    
3. 打开项目

  现在在Assets菜单，可以看到Open C# Project In Code选项。这会在VSCode打开整个项目根目录。

## 调试Unity
目前VSCode自带调试只支持通过 Mono，所以只能在Mac OS X平台使用。
不过已经有网友提供了方法：http://forum.unity3d.com/threads/vs-code-unity-debugger-extension-preview.369775/
	
- 首先下载：[unity-debug-0.5.0.zip](http://beta.unity3d.com/lukasz/vscode/unity-debug-0.5.0.zip)
- 然后解压，复制unity-debug文件夹到 %USERPROFILE%\.vscode\extensions（如果extensions不存在就新建一个）
- 重新打开VSCode，打开你的Unity项目。
- 选择Debug视图，并点击齿轮设置按钮，在弹出的菜单选择Unity Debugger。
- 如果没有这个选项，说明你没有把上面的文件夹放到正确的位置，或者你的项目中已经有了.vscode/Launch.json文件，你需要先删除它。
- 现在你的Unity项目中会有一个.vscode/Launch.json文件。
- 现在你可以在VSCode的C#文件中使用断点进行调试。在VSCode点击绿色箭头开启调试，然后再Unity中点击Play，就可以进行调试。