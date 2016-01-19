---
layout: post
title: Unity3D 向量、旋转
description: "Unity3D中 向量、旋转相关知识"
modified: 2015-09-17
tags: [Unity3D]
---

##方向和距离

一个物体到另一个物体的方向：

    //获得player位置到target位置的向量
    var heading = target.position - player.position;

距离与方向向量：

    var distance = heading.magnitude;  // 距离
    var direction = heading / distance;  // 方向向量

如果只是需要进行距离的判断，最好不要使用magnitude，因为它包含开平方运算，对CPU消耗比较大，可以换为使用sqrMagnitude ：

    if (heading.sqrMagnitude < maxRange * maxRange) {
        // Target 在 maxRange 范围内
    }

这比使用 magnitude 有效率的多。 

有时会需要水平方向的向量。比如一个地面上的palyer要接近空中的Target。如果直接用target减去player，会得到指向上的向量，而我们需要的是在地面上指向Target正下方的向量。只需要设置向量的Y分量为0即可：

    var heading = target.position - player.position;
    heading.y = 0;  // 地面上的heading.

对其他平面同理。

##计算法线/垂线向量

在mesh生成、路径跟随等很多情况，都需要用到垂直向量。
给定任意三个点，比如一个mesh的三个顶点，就很容易算出法线。只需两两相减求得两个向量，在利用向量叉乘，求出法向量：

![]({{ site.url }}/images/post/2015-09-17/Unity-VectorRotation-1.png)

    Vector3 a, b, c;
    Vector3 side1 = b - a;
    Vector3 side2 = c - a;

两个向量的叉乘会得到垂直于平面的向量，根据“左手法则”可以决定叉乘的顺序。从平面顶部向下看，垂直向量指向外（自己），则叉乘的第一个向量顺时针转向第二个向量：

    Vector3 perp = Vector3.Cross(side1, side2);

如果交换side1，side2的顺序，垂直向量将指向相反方向。

对于mesh来说，求其标准化法向量时，除了使用normalized属性，也可以用前面介绍的方法：

    var perpLength = perp.magnitude;
    perp /= perpLength;

事实证明三角形的面积等于perpLength / 2，这对于求整个mesh的平面面积或根据相对面积随机选取三角形是很有用的。


##向量大小在某一方向的分量：

当需要求一个向量在某一方向的分量大小，需要用点乘，比如求汽车向前的速度：

    var fwdSpeed = Vector3.Dot(rigidbody.velocity, transform.forward);

当然，transform.forward方向可以是任意的，但必须是单位向量。


##旋转和方向：

3D中的旋转通常使用欧拉角或者四元数。这两种方式各有优缺点。关于欧拉角和四元数更详细的内容，前面已经介绍。

我们知道，Vector既可以表示一个点（point），也可以表示一个方向（direction）（从原点测量的方向）。对应的，一个Quaternion既可以表示方位（orientation），也可以表示旋转（rotation）（从原始位置或Identity测量的旋转）。四元数测量的旋转是指从一个指向到另一个指向，并不能代表一个大于180°的旋转。

Unity3D内部所有旋转都是使用四元数的，但是为了便于理解和编辑，在监视面板显示的是对应的欧拉角。
这样的一个副作用就是，比如输入一个欧拉角值：( 0, 365, 0 )，因为四元数无法表示，所以运行后就变成了( 0, 5, 0 )。

####脚本操作：

在脚本中处理旋转时，应该使用Quaternion类和它的函数来创建和改变旋转值。有些时候使用欧拉角也是有效的，但你要时刻记住：应该使用四元数来处理欧拉角。

创建和操作四元数：
Unity的[Quaternion](http://docs.unity3d.com/ScriptReference/Quaternion.html)类有许多函数来帮助创建和操作旋转，从而避免使用欧拉角：

创建：

    Quaternion.LookRotation
    Quaternion.AngleAxis
    Quaternion.FromToRotation
        
操作：

    Quaternion.Slerp
    Quaternion.Inverse
    Quaternion.RotateTowards
    Transform.Rotate & Transform.RotateAround
	
然而有时使用欧拉角也是必要的，但在这时你要注意，必须保持你的角度为变量，并且只将它们作为欧拉角来用于旋转。虽然可能从Quaternion获取Euler角，但如果对其取值、修改或重置，会引起问题。

下面是一些应该避免的错误的例子，它们的目的是让物体绕X轴每秒旋转10度：

    //这里的问题是我们修改了一个quaternion的x的值，
    //但这个值并不代表一个角度，所以并不会产生想要的结果
    void Update () {    
        var rot = transform.rotation;
        rot.x += Time.deltaTime * 10;
        transform.rotation = rot;       
    }

    //这里的问题是，我们读取、修改并写入来自quaternion的欧拉角的值。
    //因为这些值是计算自quaternion，所以每个新的旋转都会返回迥然不同的欧拉角，
    //这可能引起万向节死锁。
    void Update () {        
        var angles = transform.rotation.eulerAngles;
        angles.x += Time.deltaTime * 10;
        transform.rotation = Quaternion.Euler(angles);
    }

下面是使用欧拉角的正确例子：

    //我们将角度值存于变量并将其应用于欧拉角，
    //但我们并未依赖读取回的欧拉值。
    float x;
    void Update () {        
        x += Time.deltaTime * 10;
        transform.rotation = Quaternion.Euler(x,0,0);
    }

####动画操作：

在许多动画编辑方案包括unity内置的动画窗口，都支持欧拉角旋转的动画过程。
这些旋转值可能经常超过四元数可表示的值。比如一个物体旋转720度，欧拉角可以表示为( 0, 720, 0 )，四元数则无法表示。

Unity动画窗口：

在Unity动画窗口，有选项允许你指定如何旋转插值 - 四元数或者欧拉角。当指定为欧拉角插值，意味着你希望执行角度指定的完整旋转。而指定为四元数时，Unity将执行最短路径的旋转到最终值。
更多参考 [Using Animation Curves](http://docs.unity3d.com/Manual/animeditor-AnimationCurves.html)。
	
外部动画资源：

对于外部动画资源，一般都包含欧拉角格式的旋转关键帧。Unity的默认操作会将其重采样，并对每一个动画关键帧都生成新的四元数关键帧，以尝试避免某些关键帧间的旋转超出四元数的表示范围。
	
比如一个动画，6帧长度从0度到270度，不重新采样直接使用四元数旋转，会直接旋转反方向90度，因为这是相反路径，如果重采样了，就会变成每一帧旋转45度，从而达到同样的效果。
	
但仍有许多情况重采样不能完全达到原始动画的效果，所以Unity有选项可以关闭animation resampling，直接使用原始动画的欧拉值。
更多参考  [Animation Import of Euler Curve Rotations](http://docs.unity3d.com/Manual/AnimationEulerCurveImport.html)。


---
> 参考：<br>
> [Vector Cookbook](http://docs.unity3d.com/Manual/VectorCookbook.html)<br>
> [Rotation and Orientation in Unity](http://docs.unity3d.com/Manual/QuaternionAndEulerRotationsInUnity.html)