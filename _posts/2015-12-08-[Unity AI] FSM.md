---
layout: post
title: AI：Uinty中Finite State Machines简介 
description: "Uinty中Finite State Machines简介"
modified: 2015-12-08
tags: [Unity3D, AI]
---

我们主要来介绍一下FSM在Unity中的实现。Unity5引入了state machine behaviors，它是Mecanim动画系统的扩展。我们可以利用这个功能来快速实现FSM系统。

### FSMs的使用
虽然我们主要使用FSM来实现游戏AI，但其实FSM可以用于许多场景。状态在现实生活中也无处不在，比如水有固体、液体、气体三种状态，这是由温度这一条件决定的，编程中就会使用温度变量来决定状态的改变。
FSM的一个特征是同一时刻只能有一个状态。但此外，一个agent可以有多个FSMs，而一个状态也可以再包含状态机。

### 使用AnimationController构造FSM
Unity5中，状态机主要还是动画系统的一部分，所以我们会使用AnimationController来实现。我们新建Unity项目，新建一个AnimationController，并打开Animator窗口。
在Aniamtor窗口，我们可以在左上角看到有Layer和Parameters窗口。Layer代表不同的层，Parameters表示参数列表，具体含义可以查看官方手册。
State machine behaviors是Unity5引进的概念，Mecanim会实现各种状态，你不用关心如何进入、转换或退出一个状态，这回通过内部逻辑来完成。

#### 创建状态
在Animator窗口可以创建状态，包括Empty，Blend Tree和Sub-State Machine。详细解释请参考官方手册。
选中l状态，点击Inspector界面的Add Behaviour按钮，输入名称，就会新建一个脚本，它继承自StateMachineBehaviour，默认会有注释的多个函数，每个函数用途都有注释说明。
我们这里暂时不需要OnStateMove和OnStateIK函数，这两个函数是针对动画的，所以先删掉。并留下OnStateEnter、OnStateUpdate、OnStateExit三个函数并取消注释。
每个函数都有Animator、AnimatorStateInfo和layerIndex三个参数。Animator就是获取当前animator controller的接口，我们要知道，state machine behavior是以asset的形式存在，而不是类的形式，因此使用这个参数是在运行时获取它引用的最好方式。AnimatorStateInfo提供了当前状态的信息。而Layer则是当前的层。

#### 设置条件
我们可以设置一些条件是AI在不同状态切换，其实就是设置参数。参数有四种类型，Int、float、Trigger、bool，然后选中状态直接的Transition，就可以设置对应参数的检测条件，比如某个参数大于5或某个参数为否等。而在脚本里，可以通过函数设置参数，从而达到切换条件的目的。

### 使用脚本实现FSM：
当然，除了使用Animator，也可以自己写脚本实现FSM，这方面网上的资料也很多。
简单的可以使用 Switch 或是if-else实现，也可以配合委托和事件来实现。

可以参考一下Unity wiki的这个实现[Finite State Machine](http://wiki.unity3d.com/index.php/Finite_State_Machine)。

