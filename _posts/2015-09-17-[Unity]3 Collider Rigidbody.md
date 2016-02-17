---
layout: post
title: Unity3D Collider，Rigidbody 和碰撞检测
description: "Unity3D中 Collider，Rigidbody 和碰撞检测简介"
modified: 2015-09-17
tags: [Unity3D]
---

## Rigidbody
 
刚体是启用一个物体物理行为的主要组件。一旦附加刚体组件，物体将收重力影响。如果刚体附加了Collider，则可以检测碰撞。
刚体可以接受力和扭矩，但Transform不能。对于刚体的运动，请不要使用Transform的属性来进行位移和旋转，而应该对物体附加力来让物理引擎自己计算结果。

#### Kinematic 
有时你需要一个物体附加了刚体组件但却不想使用物理引擎控制它（比如需要进行Trigger检测），这时你可以将Rigidbody 设置为Kinematic 。
	
#### Sleeping  
当刚体运动速度降低当一个值，即物理引擎认为它的运动停止时，它将进入“sleeping”模式，知道下次受到力或碰撞，它都将保持这个睡眠模式。这意味着睡眠模式中，并没有处理器时间分配到更新刚体状态，知道它再次激活。

大多数时候，刚体的休眠与唤醒的发生都是非常明显的。但是，当一个静态碰撞器通过改变其Transform的位置，令其移动进入刚体或移出刚体是，这个刚体可能会无法唤醒。此时，这个刚体可以使用函数 WakeUp 来唤醒。

更多请查看：[Rigidbody](http://docs.unity3d.com/Manual/class-Rigidbody.html) and [Rigidbody 2D](http://docs.unity3d.com/Manual/class-Rigidbody2D.html) 。
	
	
## Collider

Collider为物理碰撞定义了一个物体形状。包括几种基础图形碰撞：Box、Capsule、Sphere，还有Mesh Collier。另外还有Character Controller、Wheel Collider、Terrain Colllider等。
其中基本图形碰撞器效率比较高，Mesh Collider当mesh比较复杂时效率较低。

对于复杂物体的碰撞器设置，有三种方法，一种是用多个基本图形碰撞复合模拟出大概的样子，一种是直接使用Mesh Collider，还有一种是再为模型制作一个简单的Mesh，用这个mesh做Mesh Collider。

注意：
复合Colliders多作为子物体使用，但要注意RigidBody只能有一个，必须置于最顶层的父物体上。
还有，一个Mesh Collider 无法与另一个Mesh Collider产生碰撞，此时设置其中一个为convex 才行。总之最好是尽量避免使与Mesh Collider。

#### 静态碰撞：
  对于墙、地面这类不会运动的物体，可以仅添加Collider不添加Rigidbody，这称为Static Colliders。
  通常，配置好后，最好不要再重新设置静态碰撞的Transform的Position。因为这会非常影响物理引擎的性能。
  添加于有Rigidbody物体的Collider则是 dynamic colliders。
  静态碰撞物体可以与动态碰撞产生物理作用，但并不会对碰撞产生回应。
	
#### 物理材质 Physics materials
  当物体产生碰撞，它们的表面可以模拟不同的物理材质。比如冰表面很滑而橡胶球则有很大摩擦力。尽管Collider的形状在碰撞时并不会变形，但通过Physics Materials.可以设置它的摩擦力和弹力。
  更多内容查看官方手册： [Physic Material](http://docs.unity3d.com/Manual/class-PhysicMaterial.html
) and [Physics Material 2D](http://docs.unity3d.com/Manual/class-PhysicsMaterial2D.html) 。
	
#### 触发器 Triggers
  可以设置一个Collider为Trigger，触发器不受物理引擎控制，允许其他碰撞器穿过。当一个Collider穿过其中，OnTriggerEnter 事件将会触发。
	
	
## 碰撞检测

碰撞触发函数主要有两类，一类是碰撞器的，一类是触发器的。每类有三种，分别是进入、保持和离开。
OnTriggerEnter, OnTriggerStay, OnTriggerExit 是触发类消息，参数是Collider。
OnCollisionEnter, OnCollisionStay, OnCollisionExit 是碰撞类消息，参数是Collision。

下面看一下各种情况的触发条件：
首先说明一下，碰撞器的种类。

- Static Collider：静态碰撞器。<br>
    有Collider，没有Rigidbody，没有Trigger。简称**SC**。
    
- Rigidbody Collider：刚体碰撞器。<br>
    有Collider，有Rigidbody，没有Trigger。简称**RC**。

- Kinematic Rigidbody Collider：运动学刚体。<br>
    有Collider，有Rigidbody，刚体是Kinematic，没有Trigger。简称**KR**。

- Static Trigger Collider：静态触发器。<br>
    有Collider，没有Rigidbody，有Trigger。简称**ST**。

- Rigidbody Trigger Collider：刚体触发器。<br>
    有Collider，有Rigidbody，有Trigger。简称**RT**。

- Kinematic Rigidbody Trigger Collider：运动学刚体触发器。<br>
    有Collider，有Rigidbody，刚体是Kinematic，有Trigger。简称**KRT**。
	
    
Collision 碰撞检测规则：

![]({{ site.url }}/images/post/2015-09-17/Unity-Collide-1.png)

Trigger 触发检测规则：

![]({{ site.url }}/images/post/2015-09-17/Unity-Collide-2.png)