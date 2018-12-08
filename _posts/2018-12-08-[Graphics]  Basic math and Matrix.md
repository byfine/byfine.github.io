---
layout: post
title: 图形学数学基础及常用矩阵
description: "图形学向量、矩阵公式，常用变换、观察矩阵"
modified: 2018-12-08
tags: [Graphics]
---

总结一些图形学基础数学知识和常用矩阵。

## 向量点乘（dot product）：

![]({{ site.url }}/images/post/2018-12-08/mathbase-1.jpg)

#### 笛卡尔坐标系下：  
![]({{ site.url }}/images/post/2018-12-08/mathbase-2.jpg)

#### 投影（b在a上的投影）：  
![]({{ site.url }}/images/post/2018-12-08/mathbase-3.jpg)


## 向量叉乘（cross product）：

两个向量的叉乘会得到垂直于平面的向量，如图使用右手法则则方向向上：  
![]({{ site.url }}/images/post/2018-12-08/mathbase-4.jpg)

笛卡尔坐标系下：  
![]({{ site.url }}/images/post/2018-12-08/mathbase-5.jpg)


## 坐标系（coordinate frames）：

3D空间中任意3个向量，这三个向量都是单位长度，互相之间正交，互相满足叉乘关系。  
    ||u|| = ||v|| = ||w||
    u ▪ v = v ▪ w = u ▪ w = 0
    w = u x v

坐标系中任意一个向量p 可以表示为：  
    p = (p ▪ u)u + (p ▪ v)v + (p ▪ w)w


#### 通过任意向量a，b，构造坐标系 w,u,v：  
![]({{ site.url }}/images/post/2018-12-08/mathbase-6.jpg)


## 矩阵：

#### 乘法：  
![]({{ site.url }}/images/post/2018-12-08/Matrix-1.png)

满足交换律和结合律：  
A(B + C) = AB + AC 、
(A + B)C = AC + BC

#### 转置矩阵：  
![]({{ site.url }}/images/post/2018-12-08/Matrix-2.jpg)
![]({{ site.url }}/images/post/2018-12-08/Matrix-3.jpg)

#### 单位矩阵和逆矩阵：  
![]({{ site.url }}/images/post/2018-12-08/Matrix-4.jpg)


## 变换矩阵：

#### 缩放（Scale Nonuniform）矩阵 及 逆矩阵：  
![]({{ site.url }}/images/post/2018-12-08/Transform-1.jpg)

#### 错切（Shear） 及 逆矩阵：  
![]({{ site.url }}/images/post/2018-12-08/Transform-2.jpg)

#### 2D 旋转（Rotation）：  
![]({{ site.url }}/images/post/2018-12-08/Transform-3.jpg)

2D下旋转是线性的，并且可交换旋转顺序。


#### 组合旋转、缩放：

x<sub>3</sub> = Rx<sub>2</sub>,  x<sub>2</sub> = Sx<sub>1</sub>,  
x<sub>3</sub> = R(Sx<sub>1</sub>) = (RS) x<sub>1</sub>,  
x<sub>3</sub> != SR x<sub>1</sub>  

#### 组合逆变换：  
![]({{ site.url }}/images/post/2018-12-08/Transform-4.jpg)


#### 3D下 绕坐标轴旋转：  
![]({{ site.url }}/images/post/2018-12-08/Transform-5.jpg)

#### 任意轴a(x,y,z) 旋转（Rodrigues）：  
![]({{ site.url }}/images/post/2018-12-08/Transform-6.jpg)


#### 平移：  
增加一个w坐标，变为4x4矩阵：  
![]({{ site.url }}/images/post/2018-12-08/Transform-7.jpg)

#### 齐次坐标：  
一个点的非齐次坐标通过除以w得到：  
![]({{ site.url }}/images/post/2018-12-08/Transform-8.jpg)

当 w>0 表示真实的点， w=0时表示无穷远的点（一个长度为0的向量）。

使用齐次坐标，可以将所有变换统一，变换完成最后需要真实点的位置时，除以w即可。


#### 组合 旋转-平移：  
![]({{ site.url }}/images/post/2018-12-08/Transform-9.jpg)

#### 组合 平移-旋转：  
![]({{ site.url }}/images/post/2018-12-08/Transform-10.jpg)


#### 法向变换：  
t是平面的切向量，n是法向量，M是对平面使用的变换矩阵，求对n的变换矩阵Q：  
![]({{ site.url }}/images/post/2018-12-08/Transform-11.jpg)

这里 逆矩阵的转置 只针对齐次坐标左上角的 3x3 矩阵。（因为平移坐标对法向没有影响）。


#### 旋转坐标系的每一行，是新坐标系的3个单位基向量（3D同理）：  
![]({{ site.url }}/images/post/2018-12-08/Transform-12.jpg)


## 观察：  

#### 正交矩阵：  
![]({{ site.url }}/images/post/2018-12-08/viewing-1.jpg)   

![]({{ site.url }}/images/post/2018-12-08/viewing-2.jpg)  

OpenGL中观察的方向是-z，用-n、-f 代替 n、f。

#### 投影矩阵：  
d 是视点距屏幕的距离  

![]({{ site.url }}/images/post/2018-12-08/viewing-3.jpg)   

![]({{ site.url }}/images/post/2018-12-08/viewing-4.jpg)   


#### Frustum：  
![]({{ site.url }}/images/post/2018-12-08/viewing-5.jpg)   
![]({{ site.url }}/images/post/2018-12-08/viewing-6.jpg)   

ø = fov / 2,  
d = cotø,   
aspect = width / height,  

![]({{ site.url }}/images/post/2018-12-08/viewing-7.jpg)   

其中aspect用于处理不同的纵横比。  
A、B用于把 近裁剪面n 和 远裁剪面f 映射到 -1 和 1。

推导得出A、B:  
![]({{ site.url }}/images/post/2018-12-08/viewing-8.jpg)  


注意，Z的映射是非线性的，有一个-1/Z的比例项，它的优点是可以处理很广的深度范围，但是深度分辨率不统一，在接近n（近剪裁面）的时候趋于最大值。所以，不要把近剪裁面设为0。
