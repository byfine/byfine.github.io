---
layout: post
title: UGUI Layout Element 介绍
description: "UGUI Layout Element"
modified: 2019-01-21
tags: [Unity]
---

### 前言
相信对于经常使用UGUI的人来说，Layout Element 是一个刚开始非常难理解的东西，添加该组件后总是不按我们预想的那样改变 RectTransform 的大小，而且官方文档也写得比较含糊。  
其实搞明白它的原理，会发现这个东西还是很简单且实用的。

### 基本概念
自动布局系统，对于嵌套结构的UI，或者不能预先确定大小的组件是很实用的，比如 ScrollView 的 Content 自动根据内容调节大小，或是 Text 根据文字长度改变尺寸。  
自动布局都是基于 RectTransform 来构建的，可以应用于任何包含 RectTransform 的元素上。  
其中主要基于两个概念： Layout Elements 和 Layout Element Controllers。一个是用于控制每个子元素大小的，一个是用于调节所有元素的父元素的。

### Layout Elements  
在 Unity 中，除了狭义的 Layout Element 组件，每个 RectTransform 其实都有 Layout Element 的概念，我们可以在组件预览窗口查看到（要在下拉菜单选择）  
![]({{ site.url }}/images/post/UGUI-Utilities/UGUI-UT-2.jpg)   

我们可以看到其中包含下面6个属性：  
Minmum width   
Minmum height   
Preferred width   
Preferred height   
Flexible width   
Flexible height   

它们默认值都是0，一些组件添加上后会自动改变它们的值，如Text、Image等。  
除此，还有一个 Layout Element 组件，可以手动设置这些属性的值，添加上后优先度会高于其他组件。而且也可以添加多个，通过 Layout Priority 来调节互相之间的优先度。  

那么这些属性到底是干什么用的呢？

以Text为例，我们会发现尽管其改变了 Layout Priorities，但文字的组件还是我们手动设置的，和这些属性还是没关系啊。这时候就要 Layout Element Controller 登场了，只有父类添加了Controller 组件，其子类的 Layout Priorities 才会发挥作用。它们相互配合才能实现自动布局的效果。    
Layout Element Controller 包括 Horizontal Layout Group, Vertical Layout Group 和 Grid Layout Group 等组件。

下面先分别介绍一下各个属性的作用：  
    1. Minmum width/height 最先被分配，不带任何妥协。  
    2. 如果父类容器中仍有多余的空间，那么 Preferred width/height 会被分配。  
    3. 如果上面两条分配完了之后仍有额外的空间，那么 flexible width/height 会被继续分配。  
   
这样说还是难以理解，以 Horizontal Layout Group 为例来详细说明一下。

#### 实例
我们构建如下场景，其中Parent的 size 为 (100, 50)，两个 Image 为 (50, 50)，Parent 添加 Horizontal Layout Group 组件。  
![]({{ site.url }}/images/post/UGUI-Utilities/UGUI-UT-3.jpg)   

这时候观察两个子图片的 Layout Priorities，会发现 Minmum width/height 和 Preferred width/height 都是 0，如果 Layout Controller 真的生效了，为什么两张图片的大小不是0？待会解释这个问题，我们先来修改一下 Horizontal Layout Group 的设置，改为如下：   
![]({{ site.url }}/images/post/UGUI-Utilities/UGUI-UT-4.jpg)    

刚设置完你会发现两个子图片大小都变为0了，这是因为只有改为 Child Controls Size 后，Layout Controller 才会根据子物体的 Layout Priorities 对其进行设置。而此时两个图片的 Minmum width/height 和 Preferred width/height 都是 0，所以大小变为了0。  

这时我们就可以为两个 Image 添加 Layout Element 来覆盖其默认属性。 

#### Min Width/Height
首先将它们的 Min Width/Height 都设置为50，会发现图片大小都变为了 (50,50)，正好覆盖父物体大小。在调整为30，60等值，会发现不管缩小还是放到，两个子物体都不管父物体的小，只设置为其 Min Width/Height 的值，这就是  Min Width/Height 的作用，它们限制组件大小的最低标准，组件绝不会小于此值。  

#### Preferred Width/Height
接着先把 Min Width/Height 恢复为50，然后设置 Preferred Width/Height 为80（注意，Preferred 的值要比 Min 的大，因为组件尺寸无论如何都不能小于 Min 设置的值，Preferred比其小就没有意义了）。此时父物体大小是(100, 50)，接下来我们调整 Parent 的大小，会观察到一下结果：

| Parent size | Child Size |
| (100, 50) | (50, 50) |  
| (160, 80) | (80, 80) |   
| (200, 100) | (80, 80) |  
| (50, 20) | (50, 50) |  

由此可得：  
无论父容器如何变化，子物体始终不小于 Min Size。  
如果子物体在满足Min Size的情况下，父容器还有多余的空间，那么所有拥有 Preferred Size 的子类会平分多余的空间。  
当父容器空间分配给单个子物体的空间超过Preferred Size后，该子类的大小不会继续增长（仅限未设置flexible size的情况）。  

#### Flexible Width/Height
接下来了解一下 Flexible Width/Height，不同于上面两种属性直接根据值调整大小，Flexible Width/Height是根据每个子物体的相对比例来调整的。对于每个 Flexible 大于0的子物体，都会用父物体剩余的空间将其填满，而填满的大小，就是根据子物体各自 Flexible 占的比例来分配的。

为了便于观察，我们先关闭两个子物体的 Min 和 Preferred 设置，父物体大小重设为 (100, 50)，同时开启Child Force Expand。先知调整Flexible Width，不同的值可得：

| Flexible Width 1 | Flexible Width 2 | Width 1 | Width 2 |
| 1 | 1 | 50 | 50 |  
| 2 | 2 | 50 | 50 | 
| 1 | 2 | 33.33 | 66.66 |  
| 1 | 4 | 20 | 80 | 

这样就很好理解了，以第三组 Flexible 分别为 1 和 2 为例，子物体1 占了 1/3 * 100 = 33.333，子物体2 占了 2/3 * 100 = 66.666。 

下面我们在设置回 Min Width/Height 为50，Preferred Width/Height 为80，Flexible 为 1：2，这时候调整父物体宽度，当其宽度小于160（满足 Preferred 的值）时，Flexible是不会生效的。小于100时，Preferred 也不会生效（如上所述此时为Min）。而当父物体宽度大于160时，Flexible就开始生效了，比如宽度为200，可得 93.333 和 106.666 的两个子物体宽度，这是怎么来的呢？

Parent Width - (C1 Preferred Width + C2 Preferred Width) = 200 - (80 + 80) = 40  
即分配完 Preferred 的值还剩40，而这40就会根据比例再分配各两个子物体：  
C1 Width = 80 + 1/3 * 40 = 93.3333  
C2 Width = 80 + 2/3 * 40 = 106.6666  

最后说回父物体上的 Child Controls Size 和 Child Force Expand 选项，这也解释了我们之前的问题。前一个选项屏蔽掉这个Layout Controller对布局属性的影响，完全使用子容器的布局属性进行设置。后一项则是当子物体没有设置Flexibel时，将所有子容器的Flexible都设置为1，这样就可以让所有子容器以填充的方式平分父容器。


<br/>  