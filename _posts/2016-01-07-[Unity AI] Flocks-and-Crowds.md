---
layout: post
title: AI：Flocks and Crowds：群聚
description: "Flocks and Crowds：群聚及人群算法"
modified: 2016-01-07
tags: [Unity3D, AI]
---

Flocks and Crowds，这是游戏开发中一类非常有用的算法。Flocks主要指模拟非玩家角色的群体行动，如蜂群、鸟群、鱼群等；Crowds指模拟一群人物的行动，如RTS游戏中，选中一群士兵，让他们移动到某一处。

###群聚的概念

群聚算法最早由 Craig Reynolds 在八十年代提出。

群聚描述了一组物体，作为一个群体集体移动。Flocking 算法的名字来源于自然界中的鸟群（boids），鸟群中的鸟跟随其他鸟，飞向某一目的地，每一只之间都保持着几乎固定的距离。群聚研究的是，无论路径如何，一个群体中的物体都能和谐且有效率的移动。

有三个基本的概念定义了一个flocks如何运行：
    
- 分离（Separation）：在群体中和“邻居”保持距离以避免碰撞。
    
    ![]({{ site.url }}/images/post/Unity-AI/Flock-1.png)
    
    如图，中间的物体，在不改变自身指向的情况下，正在向远离其他物体的方向移动。
    
- 对齐（Alignment）：以和群体相同的方向移动，并且速度也保持一致。

    ![]({{ site.url }}/images/post/Unity-AI/Flock-2.png)
    
    如图，中间的物体，正在改变自己的指向，以匹配周围物体的方向。
    
- 凝聚（Cohesion）：与群体的中心保持最小的距离。

    ![]({{ site.url }}/images/post/Unity-AI/Flock-3.png)
    
    如图，右侧的物体，正往箭头指示方向移动，以和它周围的物体保持最小距离。
    
    
从这三条可以得知，每个单位都必须有运用转向力行进的能力。此外，每个单位都必须得知其局部周围的情况，邻近单位的位置、它们的方向、与自身的距离等。

###Unity中 Flocking 示例
接下来我们在Unity中实现一个简单的flocking示例项目。主要有两个组件：独立的boid物体脚本，和一个控制脚本来维护和领导群体。（**Boid**是 Craig Reynolds 提出的术语，指像鸟一样的物体，我们用这个术语来代表每个个体。）

![]({{ site.url }}/images/post/Unity-AI/Flock-4.png)
    
UnityFlock 表示独立的物体，这里只用简单的方块来表示，你也可以换成鸟的模型之类的。
UnityFlockController 是它们的头领，它会随机更新移动位置，而其他物体会跟着它移动。

####模仿个体行为
下面我们来实现boid的行为，创建一个UnityFlock.cs文件，用来控制每个个体的行为。

首先定义用到的属性：

{% highlight c# %} 
using UnityEngine;
using System.Collections;

public class UnityFlock : MonoBehaviour {    
   
    public float minSpeed = 20.0f;
    public float turnSpeed = 20.0f;
    public float randomFreq = 10.0f;
    public float randomForce = 10.0f;

    // 分离变量
    public float avoidanceRadius = 20.0f;
    public float avoidanceForce = 20.0f;

    // 对齐变量
    public float toLeaderForce = 50.0f;
    public float toLeaderRange = 100.0f;

    // 凝聚变量
    public float cohesionVelocity = 5.0f;
    public float cohesionRadius = 30.0f;

    // 控制boid运动的变量
    private Vector3 velocity;
    private Vector3 randomPush;
    private Vector3 leaderPush;
    private Vector3 avoidPush;
    private Vector3 centerPush;

    // 头领
    private UnityFlockController leader;
    
{% endhighlight %}

我们首先定义了最小移动速度、旋转速度，这两个速度最好和leader保持一致。*randomFreq* 和 *randomForce* 是用于随机运动的，以使运动看起来更真实。*randomFreq* 控制 *randomPush* 的更新频率，而 *randomPush* 的值根据 *randomForce* 决定。

*avoidanceRadius* 和 *avoidanceForce* 用来保持个体间的最小距离，这些属性是用来实现“分离”原则的。

*toLeaderRange* 指示群体扩散的范围，*toLeaderForce* 保持boid在范围内并且与头领保持一定距离。用来实现“对齐”原则。 

*cohesionVelocity* 和 *cohesionRadius* 用来保持和群体中心的最小距离，以实现“凝聚”原则。

*leader* 物体就是父物体，来控制整个群体的运动。每个boid还需要知道其他boid的运动情况，所以，leader中会有数组 *Flocks* 保存所有boid的信息。


接下来是start函数，指定父物体为 *leader*，且开启了更新随即运动的协程：

{% highlight c# %} 
void Start ()
{
    randomFreq = 1.0f / randomFreq;

    if (transform.parent){
        // 指定父物体为leader
        leader = transform.parent.GetComponent<UnityFlockController>();
    }

    if (leader.Flocks != null && leader.FlockCount > 1){
        transform.parent = null;
        // 开始 计算随机移动 的协程
        StartCoroutine(UpdateRandom());
    }
}
{% endhighlight %}


随即运动协程函数：

{% highlight c# %} 
IEnumerator UpdateRandom ()
{
    while(true)
    {
        randomPush = Random.insideUnitSphere * randomForce;
        yield return new WaitForSeconds(randomFreq + Random.Range(-randomFreq / 2.0f, randomFreq / 2.0f));
    }
}
{% endhighlight %}


接下来是update()函数，分别实现三个原则，计算速度，并应用速度进行位移和旋转。

####实现控制器
控制器就是leader，它的作用就是随机移动，并保存所有群体个体数组。

这里就不贴代码了，代码详情请查看 [github 项目源码](https://github.com/byfine/Unity-AI---Flocks-and-Crowds)，里面有详细注释。

###另一种Flocking算法实现方式
这是一种更简单的实现flocking算法的方法。我们使用Unity中的rigidbody到我们的boid上，通过刚体物理，我们可以简化运动控制。为了避免boid互相重叠，可以添加球形碰撞器。

和上面例子一样，还是需要两部分：个体和控制器。所有个体跟随控制器运动。

在FlockController中，我们会生成所有物体，然后计算群体的中心点和平均速度。并且定义了控制三个原则的变量。
然后再每个Flock中，我们应用FlockController计算出的中心点、平均速度和各个控制变量，连计算运动的速度，并施加于刚体上。

代码详情请查看 [github 项目源码](https://github.com/byfine/Unity-AI---Flocks-and-Crowds)，里面有详细注释。

如果需要运动的更具真实性，可以随机改变 群聚度、分离度等属性的值。