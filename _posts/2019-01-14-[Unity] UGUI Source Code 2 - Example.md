---
layout: post
title: UGUI源码分析 2 - 从一个简单例子了解UI绘制和点击事件
description: "UGUI源码，简单例子"
modified: 2019-01-14
tags: [Unity]
---

## 绘制和点击
UI模块的主要两大组成就是绘制和交互。  
绘制的本质其实和绘制模型一样，就是对 2D mesh 进行渲染，UI图片就是贴图，设置成sprite只是为了方便打包图集及进行其他UI相关操作，而对图集图片的显示就是通过UV的控制。  
点击的本质跟简单， 其实就是射线检测，摄像机发射射线，对是否点击到UI layer的物体进行检测。
据此，我们写一个简单例子了解一下。

## 代码实例

####创建物体
创建一个空物体，添加 MeshFilter、MeshRenderer 组件用于绘制，添加 MeshCollider 用于点击检测。

####绘制mesh
{% highlight c# %}  
public Mesh mesh;
public MeshFilter meshFilter;
public VertexHelper vertexHelper;

void InitMesh()
{
    mesh = new Mesh();
    vertexHelper.Clear();
    // 添加矩形四个顶点
    vertexHelper.AddVert(new Vector2(0, 0), Color, new Vector2(0, 0));
    vertexHelper.AddVert(new Vector2(0, 1), Color, new Vector2(0, 1));
    vertexHelper.AddVert(new Vector2(1, 1), Color, new Vector2(1, 1));
    vertexHelper.AddVert(new Vector2(1, 0), Color, new Vector2(1, 0));
    // 指定绘制三角形
    vertexHelper.AddTriangle(0, 1, 2);
    vertexHelper.AddTriangle(2, 3, 0);
    // 填充mesh
    vertexHelper.FillMesh(mesh);

    meshFilter.mesh = mesh;
    meshCollider.sharedMesh = mesh;
}
{% endhighlight %}   

VertexHelper 是Unity UI 的一个辅助类，可以帮助创建mesh等相关操作。利用此类，创建了一个四个顶点的矩形面片，然后设置两个三角形，以规定绘制的顺序，接着就是填充面片，设置参数。

接着实时更新 MeshRenderer 的参数，这样就能绘制出对应的颜色和图片。

{% highlight c# %}
public Color Color;
public Texture2D Texture;

// 在 Update函数调用
void UpdateRenderer()
{
    meshRenderer.material.color = Color;
    meshRenderer.material.mainTexture = Texture;
}
{% endhighlight %}

####点击检测
点击检测很简单，就是利用摄像机发出的射线实时检测。

{% highlight c# %}
public Camera Cam;    
// 在 Update函数调用
void ClickCheck()
{
    Ray ray = Cam.ScreenPointToRay(Input.mousePosition);
    RaycastHit hitInfo;
    if (Physics.Raycast(ray, out hitInfo))
    {
        if (Input.GetMouseButtonDown(0))
        {
            Debug.Log("click!");
        }
    }            
}
{% endhighlight %}

####总结
可以看到这个例子里，两个功能都实现的很简陋，UGUI真正实现肯定没这么简单，但是通过层层分析源码，可以发现最后实现的本质就是这样，理解这个例子对我们后面学习源码有很大帮助。


