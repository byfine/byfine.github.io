---
layout: post
title: Scriptable Objects 及 游戏架构
description: "Scriptable Objects 相关介绍，及基于其的游戏架构技术"
modified: 2019-01-31
tags: [Unity]
---

### 什么是 ScriptableObject？
ScriptableObject 是一个可序列化的Unity类，能够让你在脚本实例之外存储大量共享数据。ScriptableObject 可以避免复制大量数值从而减少内存。使用脚本化对象可以更轻松地管理更改和调试。不同于使用Prefab挂载脚本保存数据，使用时实例化多个。使用 ScriptableObject 存储数据，可通过引用从任何Prefab访问数据，而内存中只有一个数据副本。

如 MonoBehaviours 一样，ScriptableObject 也继承自 Unity Object。但不同于 MonoBehaviours，不能把 ScriptableObject 附加于 GameObject 上，相反而是当做 资产 存放与项目内。

使用Editor时，可在编辑时和运行时将数据保存到 ScriptableObjects，因为 ScriptableObjects 使用 Editor命名空间 和 Editor脚本。但是，在已发布的构建中，不能向 Scriptable Objects 保存数据，但可以在运行期间读取其中保存的数据。编辑器工具保存到 ScripableObjects 资产的数据会写入磁盘，因此在会话之间是持久的。

### 使用 ScriptableObject
要使用 ScriptableObject，需要先在Asset目录下创建一个脚本，然后让其继承 ScriptableObject类。 可以使用[CreateAssetMenu]() 属性来快速创建自定义资源，比如：

{% highlight c# %}  
using UnityEngine;

[CreateAssetMenu(fileName = "Data", menuName = "ScriptableObjects/SpawnManagerScriptableObject", order = 1)]
public class SpawnManagerScriptableObject : ScriptableObject
{
    public string prefabName;

    public int numberOfPrefabsToCreate;
    public Vector3[] spawnPoints;
}
{% endhighlight %}  

你现在可以通过 Assets > Create > ScriptableObjects > SpawnManagerScriptableObject 来创建该 ScriptableObject。

也可以通过脚本创建：

{% highlight c# %}  
[MenuItem("Assets/Create SO")]
public static void CreateSO(){
    var asset = ScriptableObject.CreateInstance<SpawnManagerScriptableObject>();
    AssetDatabase.CreateAsset(asset, "Resources/ScriptableObjects/mySO.asset");
    AssetDatabase.SaveAssets();
    AssetDatabase.Refresh();
}
{% endhighlight %}  

要在运行时使用SO的数据，只要在自定义类中引用该 SO 类型，并在Editor拖动赋值或通过脚本获得该 SO资产 即可。


### 项目工程的三大支柱
下面我们介绍一下使用 ScriptableObject 优化游戏结构的一些方法，在此之前，先了解一些基础理论知识。首先介绍下项目工程的三个支柱：

+ 模块化  
  + 避免创建直接相互依赖的系统。例如，库存系统应该能够与游戏中的其他系统通信，但最好不要在它们之间创建硬引用，因为这会使得重组织系统的配置和关系变得很困难。
  + 从头开始创建场景：避免在场景之间存在瞬态数据。每当点击一个场景时，场景都应是干净独立的关闭和加载。这可以保证你的每个场景都有其他场景没有的唯一行为。
  + 设置Prefab，使其独立工作。尽可能地保证，拖入场景的每个Prefab其功能都只在其内部实现。这有助于大型团队进行源码控制，其中场景就是一堆Prefab组成的列表，而每个Prefab包含单独的功能。这样，您的大部分操作都处于Prefab级别，从而减少了场景中的冲突。
  + 让每个组件专注于只解决一个问题。这样可以更轻松地将多个组件拼凑在一起以构建新的组件。

+ 可编辑  
  + 尽可能多利用数据驱动游戏：当你将游戏系统设计成好像 将数据作为指令处理的机器一样时，即使在游戏运行期间，也能高效地对游戏进行更改。
  + 如果尽可能将系统设置为模块化和基于组件的，系统将更容易进行编辑，即使对于美术和策划来说。如果策划能在游戏中拼凑东西，而无需提出明确功能需求，这在很大程度上要归功于实现了每个小组件只做一件事，那么就可以不同的方式组合这些组件，以找到新的游戏玩法或机制。
  + 团队可以在运行时更改游戏是非常重要的。运行时可对游戏进行的更改越多，就可越多地找到平衡和数值。如果还能够将运行时状态保存回来（比如 ScriptableObject），那就更好了。

+ 可调试  
  + 这更像前面两个的子支柱。游戏模块化程度越高，就越容易测试其中的任何一个部分。也就是说，游戏越是可编辑，在Inspector视图中拥有的功能就越多，调试也就越容易。请确保你可以在Inspector中查看调试状态，并且实现一个功能时，在没有调试计划之前永远不要考虑去完成它。

### Singleton 的利弊
Singleton 相信大家都不陌生了，应该是最常用也最简单的一种模式。我们来探讨一下它的利弊：

利：

+ 可以从任何地方获取任何东西
+ 持久化状态
+ 便于理解
+ 统一的模式
+ 便于“计划”

虽然优点不少，但滥用Singleton对项目结构来说是很不好的，Singleton 问题有：

+ 固定的连接，非模块化
+ 没有多态性
+ 不可测试
+ 依赖关系混乱
+ 单一实例

解决的办法： 

+ 减少全局管理器的需求
+ 控制反转
  + 比如：依赖注入
  + 对象被赋予依赖关系
  + 单一责任原则

### 模块化数据

#### ScriptableObject 变量

先考虑一个例子，现在有一个 Enemy Prefab，它需要知道自身的移动速度，你可以直接在Prefab上修改其 MoveSpeed 属性，但是假如我们有20个 Enemy Prefab，每个都有一些区别但它们移动速度是一样的，那你要为每个都赋值一遍。  
一个解决方法是通过一个 Singleton来获取： ``EnemyManager.Instance.MoveSpeed``。  
但这个方法有上述我们说过的缺点： 这是硬连接；这个属性完全依赖于加载的 Manager。比如我们要测试敌人的速度，就要完全从头运行场景，等待 Manager 加载配置完成，然后拖入 Prefab 测试，而不能直接在一个测试场景快速的拖入Prefab直接测试。   
这时候你可能会想到使用 ScriptableObject，创建一个 EnemyConfig 的 ScriptableObject，其中保存敌人的配置数据，20个Prefab都可以获取这个引用读取数据。但这也有一些问题： 会将敌人各部分属性都混到一起；限制扩展性和模块化。比如这时候有一个敌人是范围攻击，它需要 MinFireRange 和 MaxFireRange 两个属性，这是两个新属性，加入EnemyConfig的话会产生冗余，因为其他敌人并用不到，不加入的话又打破了模块化和扩展性。

下面引入 ScriptableObject 变量 的概念。

ScriptableObject 用于架构游戏的最简单的方法之一是，自包含的基于资源的变量。以下是FloatVariable的示例，但该变量也可扩展到任何其他可序列化类型。
{% highlight c# %}  
[CreateAssetMenu]
public class FloatVariable : ScriptableObject
{
    public float Value;
}
{% endhighlight %}  

使用该变量，团队中的每个人无论技术如何，都可以通过创建一个 新的 FloatVariable Asset 来定义新游戏变量。任何 MonoBehaviour 或 ScriptableObject 都可以使用 public FloatVariable（而不是 public float），以引用这个新的共享值。

更好的是，如果一个 MonoBehaviour 改变了 FloatVariable 的值，其他的M onoBehaviour 就可以看到这个改变。这在系统之间创建了一种消息传递层，从而不需要彼此引用。

一个示例情况就是，玩家的HP。游戏中有一个本地玩家，其HP值是名为 PlayerHP 的 FloatVariable。当玩家受伤，PlayerHP减少，治疗时增加。现在想象场景中有一个 生命指示条，实时更新显示玩家的HP。不需更改任何代码，它可以轻松地指向不同的对象，如PlayerMP变量。生命指示条对场景种的玩家一无所知，它只是一直读取玩家写入的一个变量。

![]({{ site.url }}/images/post/ScriptableObject/SO1.png)

一旦这样设置，我们就可以简单的增加更多观察 PlayerHP 的系统。如音乐系统可以在PlayerHP过低时改变BGM，敌人可能改变攻击策略当检测到玩家血量过低。其中的关键就是，Player 不需要向这些系统发送消息，而这些系统也不需要知道 Player GameObject的信息。你也可以在游戏运行时，在Inspector修改 PlayerHP 的值来进行测试。

编辑FloatVariable的值时，最好是将数据复制到运行时值，而不更改存储在磁盘上的ScriptableObject的值。如果这样做，MonoBehaviour应访问RuntimeValue，以防止编辑了保存到磁盘的初始值。 
{% highlight c# %}  
[CreateAssetMenu]
public class FloatVariable : ScriptableObject, ISerializationCallbackReceiver
{
    public float InitialValue;
    [NonSerialized]
    public float RuntimeValue;

    public void OnAfterDeserialize()
    {
        RuntimeValue = InitialValue;
    }
    
    public void OnBeforeSerialize(){}
}
{% endhighlight %}  
<br/>

#### 事件  
事件架构可在彼此不了解的系统之间发送消息来帮助模块化代码。它们允许事件对状态的更改作出响应，而无需在Update循环中进行持续监控。

Unity有 UnityEvent，可以方便的实现事件系统，我们直接在Inspecor就可以对其赋值。但很明显它也有很多问题：固定的绑定（如一个按钮事件硬引用它要响应的对象）；有限的序列化；垃圾分配。所以我们要使用自己的Events。

你可以通过 ScriptableObject 构建一个 Event 系统，包含两个部分：GameEvent ScriptableObject 和 GameEventListener MonoBehaviour。可以在项目中创建任意数量的GameEvent来表示可以发送的重要消息。GameEventListener等待特定GameEvent被引发，并通过调用UnityEvent作出响应（这不是一个真正的事件，而是序列化函数调用）。

{% highlight c# %}  
public class GameEvent : ScriptableObject
{
    private readonly List<GameEventListener> listeners = new List<GameEventListener>();

    public void Raise()
    {
        for (int i = listeners.Count; i >= 0; i--)
        {
            listeners[i].OnEventRaised();
        }
    }

    public void RegisterListener(GameEventListener listener)
    {
        listeners.Add(listener);
    }

    public void UnregisterListener(GameEventListener listener)
    {
        listeners.Remove(listener);
    }
}

public class GameEventListener : MonoBehaviour
{
    public GameEvent Event;
    public UnityEvent Response;

    private void OnEnable()
    {
        Event.RegisterListener(this);
    }

    private void OnDisable()
    {
        Event.UnregisterListener(this);
    }

    public void OnEventRaised()
    {
        Response.Invoke();
    }
}
{% endhighlight %}  

一个实例是，在游戏中出来玩家死亡。在这一点上，很多执行都可对其更改，但很难确定具体在何处对所有逻辑进行编码。Player脚本是否应该触发 游戏结束UI、改变音乐？如果玩家还处于活着状态，敌人是否应该每一帧进行检查？事件系统可让我们避免类似这种有问题的依赖关系。

如果玩家死亡，Player脚本会在 OnPlayerDied事件 上调用 Raise。Player脚本不需要知道哪些系统与之相关，因为它只是一个广播。游戏结束UI 侦听 OnPlayerDied事件 并开始显示动画，摄像机脚本可以侦听它并开始淡化为黑色，音乐系统可以作出响应来改变BGM。我们也可以让每个敌人侦听 OnPlayerDied，触发讽刺动画或更改状态机到Idle。

这种模式可以非常容易地为玩家死亡添加新的响应。此外，还可以通过从Inspector中的某些测试代码或按钮事件调用Raise，轻松地对玩家死亡响应进行测试。

![]({{ site.url }}/images/post/ScriptableObject/SO2.png)


#### 以Asset为基础的系统
ScriptableObject 不一定只是数据。你可以考虑把任何你在 MonoBehaviour 中实现的系统转移到 ScriptableObject 中实现。相比于在 DontDestroyOnLoad 的 MonoBehaviour 上实现 InventoryManager，试试将其放在 ScriptableObject 上。

因为它不绑定于场景，所以不需要Transform也不需要Update函数，就可以在各个场景间维持状态，也不需要任何场景初始化设置。相比于 Singleton，当一个脚本在需要获取Inventory，只要使用一个public参数引用 Inventory Object 即可。将使得更换 测试Inventory 还是 教程Inventory 都变得很容易。

可以想像一个引用 Inventory系统 的Player脚本。当玩家生成时，它可以向库存询问所有拥有的对象并生成任何装备。装备UI 也可以引用这个 Inventory系统 并循环遍历物品以确定要绘制的内容。

### Github实例
[https://github.com/roboryantron/Unite2017](https://github.com/roboryantron/Unite2017)

[https://github.com/DanielEverland/ScriptableObject-Architecture](https://github.com/DanielEverland/ScriptableObject-Architecture)

[https://github.com/Begounet/scriptableobjects-oriented](https://github.com/Begounet/scriptableobjects-oriented)