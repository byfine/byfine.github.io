---
layout: post
title: AssetBundle 最佳实践
description: "AssetBundle 最佳实践"
modified: 2019-01-29
tags: [Unity]
---

## 1. 资产、对象和序列化

### 1.1 资产和对象  
了解Unity如何识别和序列化数据，对于正确地管理数据非常重要，而其中的关键就是 Assets 和 UnityEngine.Objects 的区别。  

+ Asset 是存储在硬盘，位于Unity项目Asset目录下的文件。 如 贴图、模型、音频文件等。  
+ Object 是一组序列化的数据，是共同描述某一资源的特定实例。其描述的可以是Unity使用的任意类型资源，所有Objects都是 UnityEngine.Object 的子类。  

虽然大部分的Object类型都是内置的，但是有两类特殊类型：  

+ ScriptableObject：开发者可实现自定义数据类型。这些类型可以通过Unity进行本地序列化和反序列化，并在Unity Editor中进行操作。   
+ MonoBehavior：提供了一个链接到MonoScript的封装器。MonoScript 是一种内部数据类型，用来在特定程序集和命名空间内保存对特定脚本类的引用。MonoScript 不包含任何实际的可执行代码。  

资产和对象之间存在一对多关系；也就是说，任何给定的资产文件都包含一个或多个对象。

### 1.2 内部对象引用
所有对象都可以包含对其他对象的引用。如一个材质对象包含一或多个对贴图文件的引用。  
序列化时，这些引用由两部分独立数据构成： File GUID 和 Local ID。GUID 标识存放目标资源的资产文件。本地唯一的ID标识资产文件中的每个对象，因为资产文件可能包含多个对象。  
GUID存储于 .meta文件中。打开 .meta文件开头即可看到：    
{% highlight c# %}  
fileFormatVersion: 2
guid: 49f36dba4ddaa42399594d6630b143f6
...
{% endhighlight %}  

用文本编辑器打开某个文件如一个material，可以看到：  
{% highlight c# %}  
--- !u!21 &2100000
Material:
  serializedVersion: 6
  ...
{% endhighlight %}  

其中 &开头的 &2100000 就是Local ID。   
假设有一个Cube使用了这个Material，并制作了一个Prefab，用文本编辑器打开该Prefab文件，可以看到：   
{% highlight c# %}  
MeshRenderer:
  ....  
  m_Materials:
  - {fileID: 2100000, guid: 49f36dba4ddaa42399594d6630b143f6, type: 2}
  ....
{% endhighlight %}  


### 1.3 为什么使用GUID和本地ID
使用这种机制，可以提升健壮性，并提供一个灵活且与平台无关的工作流程。

文件的GUID提供文件特定位置的抽象表达，只要有与特定文件关联的GUID，那么这个文件在磁盘上的位置就无关紧要了，该文件可以自由移动，而无需更新引用该文件的对象。   
而由于资产文件可能包含多个 UnityEngine.Object 资源，因此，需要本地ID来明确区分每个不同的对象。   

如果GUID与资产文件的关联丢失，那么对该资产文件中的所有对象的引用也将丢失，所以要保证 .meta 文件与资产同名并在同一文件夹内。   

Unity编辑器拥有 已知文件GUIDs 和 资源文件路径 的映射。如果Unity编辑器在打开状态下，meta文件0丢失，但是资源路径没有更改，编辑器可以确保保留相同的GUID。  
如果在Untiy编辑器关闭时丢失了meta文件，或者资产的路径发生变化，而meta文件没有和资产一起移动，那么所有对该资产中对象的引用都会丢失。

### 1.4 复合资产和导入器
非原生类型的资产必须导入Unity才能使用，这是通过资产导入器完成。可以使用API [AssetImporter](http://docs.unity3d.com/ScriptReference/AssetImporter.html) 在脚本手动调用。

导入的结果会产生一个或多个UnityEngine.Objects。如导入一张PNG图集，会生成一个父资产的多个自资产，每个对象共享一个文件GUID，因为它们的源数据储存在相同的资产文件中，而使用本地ID可以将其区分。

导入会把资源转换为适合平台的格式，可能还会包括许多耗时的操作，例如纹理压缩，因为这个过程很长，所以导入的资源会缓存在项目根目录下的 Library/metadta 文件夹下（以文件GUID前两位命名），无需在下次启动时重新导入。

### 1.5 序列化和实例化
虽然GUID 和 Local ID 系统健壮性强，但 GUID 对比比较慢，所以Unity 内部会维护一个缓存（PersistentManager），将 GUID 和 Local ID 转换为简单唯一的整数，即 Instance IDs，当新对象注册到缓存时以简单的单调递增方式分配ID。

缓存会维护给定的Instance ID、GUID、本地ID，和内存中对象的实例（如果有的话）之间的映射。这允许UnityEngine.Objects健壮的维护彼此的引用，解析Instance ID引用可以快速返回由Instance ID表示的加载对象，如果目标对象尚未加载，则可以将文件GUID和本地ID解析为对象的源数据，从而使Unity能够及时加载对象。   

启动时，Instance ID 缓存会使用项目马上需要的 Objects 及 Resources 文件夹内 Objects 的所有数据初始化。当运行时有新资源加入（如代码创建贴图）及 AB 加载对象时，会在缓存中增加记录。只有当 对特定文件 GUID 和 Local ID 提供使用权的AB被卸载时，Instance ID 记录会被删除。相应的映射也被删除以节省内存，如果重新加载 AB，则将重新为其加载的每个对象创建新的 Instance ID。

在特定平台上，一些事件可能导致 Objects 从内存移除。如 iOS上，当APP挂起 图形资源会被从显存卸载。如果加载这些 Objects 的 AB已经被卸载，那么Unity将不能重新为这些 Objects 加载源数据。任何对这些 Objects 现有的引用也将失效。这会导致场景中出现网格不可见或紫色纹理。

### 1.6 MonoScripts
需要理解 MonoBehaviour 包含对 MonoScript 的引用，并且 Monoscript 仅包含了定位特定脚本类所需的信息。这两种 Ojbect 类型都不包含可执行脚本类。

MonoScript包含三个字符串：程序集名，类名 和 命名空间。

当构建项目时，Unity会将Assets文件夹中的所有分散的脚本文件，编译为Mono程序集。这些程序集以及预构建的程序集DLL文件都包含在Unity最终包中。他们也是MonoScript引用的程序集，与其他资源不同，包含在Unity中的所有程序集都在引用程序启动时加载（StreamingAsset下的程序集除外）。

MonoScript Object 的存在，让 AssetBundle（或 Scene、prefab）中的  MonoBehaviour 组件 并不包含可执行代码。这样保证了 不同的 MonoBehavior 引用特定的共享类，即使这些 MonoBehavior 位于不同的AssetBundle中。

### 1.7 资源生命周期
一个 Object 会自动加载，当：  

+ 映射到该对象的 Instance ID 被间接引用   
+ 该对象目前还没有加载到内存中   
+ 对象的源数据可以被定位   

Objects 也可以使用脚本显示加载。当 Object 被加载，Unity会尝试通过将每个引用的文件GUID 和 本地ID 转换为 Instance ID 来解析任何引用。如果两个条件为真，则一个 Ojbect 的 Instance ID 第一次被引用时，其将被按需加载：  

+ Instance ID 引用的对象当前未加载        
+ Instance ID 有有效的 文件GUID 和 本地ID 注册在缓存中    

这个情况通常发生在引用本身被加载并解析之后的很短时间内。

如果 文件GUID 和 本地ID 没有 Instance ID，或者如果一个 已卸载的对象的 Instance ID 引用了无效的 文件GUID 和 本地ID，那么引用被保留，但实际的对象不会被加载。这在Unity编辑器中显示为 Missing，在运行的时候，或在场景视图中，Missing对象将以不同的方式可见，例如，网格丢失，会不可见，纹理丢失会显示紫色等。

在三种特定情况下，Object 会被卸载：  

+ 当 Unused Asset 被清理时 Ojbects 会被自动卸载。这个过程会在以下情况自动触发，场景强制切换（非递增地调用 SceneManager.LoadScene），或是 脚本调用 Resources.UnloadUnusedAssets。这个过程只卸载未被引用的 Objects，没有 Mono变量 或 其他 Objects 引用它。另外，被标志为 HideFlags.DontUnloadUnusedAsset 和 HideFlags.HideAndDontSave 的对象也不会被卸载。   
+ 来自于Resources的 Objects 可以被 Resources.UnloadAsset 显示地卸载。这些 Objects 的 Instance ID 仍然有效，并且依然包含有效的文件GUID和本地ID。如果有变量或其他对象引用了 Resources.UnloadAsset 卸载的对象，一旦这些引用被间接引用，对象将再次被加载。  
+ 来自于 AssetBundles 的 Ojbects 通过调用 AssetBundle.Unload(true) 会马上被自动卸载。这会使对象 Instance ID 的 文件GUID 和 本地ID 变为无效，任何引用卸载的 Objects 的引用将变为 missing。  

如果调用了AssetBundle.Unload(false)，那么来自这个 AssetBundle 的 Objects 不会被卸载，但Unity会使其 Instance ID 的 文件GUID 和 本地ID 无效。如果这些对象从内存中卸载，但是有其他对象对它的引用，那么Unity是无法重新加载这些对象的（最常见的情况是，在运行时将对象从内存中移除而未被卸载时，Unity会失去对其图形上下文的控制权。 当移动应用程序被暂停并且该应用程序被强制置于后台时，可能会发生这种情况。 在这种情况下，移动操作系统通常会从GPU内存中清除所有图形资源。 当应用程序返回到前景时，Unity必须在场景渲染恢复之前将所有需要的纹理，着色器和网格重新加载到GPU）。

### 1.8 加载大层次结构
在序列化 Unity GameObjects 的层次结构时，如在Prefab序列化期间，整个层次结构将完全序列化。也就是说，层次中的每个 GameObject 和 Component 将分别在序列化数据中描述。这对加载和实例化 GameObject 的层次结构所需的时间有很大影响。

在创建任何GameObject层次结构时，CPU时间用于几种不同的方式：   

+ 读取源数据（来自存储设备，来自AssetBundle，来自另一个GameObject等）   
+ 在新的Transform之间设置父子关系    
+ 实例化新的GameObjects和组件    
+ 在主线程中唤醒新的GameObjects和组件   

后三种时间成本通常是不变的，无论该层次结构是从现有结构克隆的，还是从存储加载的。但是，读取源数据的时间会随着 组件和GameObjects 序列化到层次结构中的数量的增加而线性增加，并且还要乘以数据源的加载速度。

在目前所有的平台上，从内存中的其他位置读取数据比从存储设备加载数据要快得多。此外，存储介质的性能表现在不同平台之间差异很大。因此，在低速存储设备的平台上加载Prefab时，从存储中读取Prefab序列化数据的时间可能会大大超过实例化Prefab的时间。也就是说，加载操作的成本与存储 I / O 时间有关。

如上所述，当序列化一个整体的Prefab时，每个GameObject和组件的数据都会被单独序列化，这可能会使数据重复。比如，一个有30个相同元素的UI界面将会序列化相同元素30次，产生大量大量二进制数据。加载时，这30个重复元素中，每一个元素的GameObjects和组件数据必须在传输到新实例化对象之前从磁盘读取，增加读取时间。

对于大层次结构，应该将其模块化，分别实例化，然后运行时再组装起来。

Unity 5.4注意：Unity 5.4改变了内存中transforms的排列。每个根transforms的整个子层次结构都存储在紧凑，连续的内存区域中。在实例化新的GameObjects时，这些新的GameObjects会立即重新排列到另一个父层次结构中，请考虑使用带有parent参数的 GameObject.Instantiate 重载变体。使用此重载可以避免为新的GameObject分配根transform 结构。在测试中，这加快了实例化操作所需的时间约5-10％。    
<br/>


## 2. Resources文件夹

### 2.1 Resources 系统最佳实践
<b>不要使用它！</b>    
这个建议主要有以下几个原因：   

+ 使用Resources文件夹会使细粒度的内存管理变得更加困难。   
+ Resources 文件夹的使用不当会增加程序启动时间和构建时间。      
+ 随着Resources文件夹数量的增加，管理其中的资源将会变得更困难。   
+ Resources 系统降低了项目传递自定义内容到特定平台的能力，并且无法增量更新内容。
+ AssetBundle变体是Unity用于按设备调整内容的主要工具。

### 2.2 正确使用 Resources 系统
在不妨碍良好的开发实践前提下，Resources系统在两种用途下是有益的：   

+ Resources文件夹的简易性可以用来快速构建项目原型。但是，当项目进入全面生产阶段时，应取消使用Resources文件夹。    
+ Resources文件夹在一些琐碎的案例下也有用处，如果其内容：   
+ 在整个游戏的生命周期都需要使用    
+ 内存压力不大    
+ 不需要补丁更新，或者在不同平台之间不需要单独处理    
+ 需要构建最小包   

第二种情况的例子包括用于保存 Prefab 的 MonoBehaviour单例，或者包含第三方配置数据（比如Facebook应用程序ID）的 ScriptableObjects。

### 2.3 Resources序列化
在构建工程的时候，名为Resources的所有文件夹中的 Asset 和 Object 都合并为一个序列化文件。这个文件还包含 metadata 和 索引信息，与 AssetBundle 类似。此索引包含了一个序列化的查找树，用于将给定 Object 的名字解析为适当的 文件GUID和本地ID，它也用于在序列化文件中根据特定字节偏移定位对象。

在大多数平台上，查找数据结构都是使用平衡搜索树，其查找时间以 0(n log(n)) 的速率增长，随着Resources文件夹中对象数量的增加，这种增长也会导致索引的加载时间增长超过线性增长。

此操作是不可跳过的，发生在应用启动的时候，这个时候应用会显示splash界面，根据观察，初始化一个包含10000个资产的资源系统在低端移动设备上会耗时很多秒，就算是Resources文件夹中包含的大多数对象不会加载到第一个场景中。   
<br/> 


## 3. AssetBundle 基本原理  

### 3.1 综述  
AssetBundle系统提供了一种方法来 存储一个或多个文件到一个Unity可以索引和序列化的档案格式。AssetBundles 是用于在应用安装后，为非代码内容提供加载或更新的主要工具。 这允许开发人员提交更小的应用程序包，减少运行时内存压力，并有选择地加载针对用户设备优化的内容。

### 3.2 AssetBundle 设计
总的来说，一个AssetBundle由两部分组成：头部和数据段。

头部包含有关AssetBundle的信息，例如标识符，压缩类型和清单(manifest)。清单是一个由 Object名 作为 key 的查找表。每个记录都提供一个字节索引，用于指示在AssetBundle数据段中在哪里找到给定的 Object。在大多数平台上，这个查找表被实现为一个平衡搜索树。Windows和OSX派生的平台（包括iOS）采用红黑树。因此，构建清单所需的时间将随着AssetBundle内资产数量的增长而线性增加。

数据段包含通过序列化AssetBundle中资产而生成的原始数据。如果将LZMA指定为压缩方案，则所有序列化资产的完整字节数组将被压缩。如果指定LZ4，每个资源的字节将被单独压缩。如果不使用压缩，数据段将保持为原始字节流。

在Unity 5.3之前，对象无法在AssetBundle中单独压缩。因此，如果用5.3版本的Unity版本从压缩的AssetBundle中读取一个或多个对象，那么Unity必须解压缩整个AssetBundle。通常，Unity会缓存AssetBundle的解压的副本，以提高同一AssetBundle上后续加载请求的加载性能。

### 3.3 加载AssetBundle
AssetBundles可以通过四个不同的API加载。 这四个API的具体行为取决于两个条件：    

+ AssetBundle是否是LZMA压缩，LZ4压缩或未压缩   
+ AssetBundle正在加载的平台    

这些API是：

+ AssetBundle.LoadFromMemory（可选异步）  
+ AssetBundle.LoadFromFile（可选异步）  
+ UnityWebRequest 的 DownloadHandlerAssetBundle   
+ WWW.LoadFromCacheOrDownload（在Unity 5.6或更早版本上）
  
#### 3.3.1 AssetBundle.LoadFromMemory(Async)
Unity的建议不要使用这个API。

AssetBundle.LoadFromMemoryAsync 从托管代码的字节数组（C＃中的byte[]）中加载一个AssetBundle。它总会把 来自托管代码字节数组的源数据 拷贝到 新分配的连续内存块中。如果AssetBundle是LZMA压缩的，它将在复制时解压缩AssetBundle。未压缩的和LZ4压缩的AssetBundles将被逐字复制。

此API所需的最大内存量至少为AssetBundle的两倍：由API创建的一个副本位于本地内存中，以及托管字节数组中的一个副本用于传递给API。因此，通过此API创建的AssetBundle在加载资产时，将在内存中复制三次：一次位于托管代码字节数组中，一次位于AssetBundle的本地内存副本中，第三次位于GPU或系统内存中用于资产本身。

在Unity 5.3.3之前，这个API被称为AssetBundle.CreateFromMemory。它的功能没有改变。

#### 3.3.2. AssetBundle.LoadFromFile(Async)
AssetBundle.LoadFromFile 是一个高效的API，用来从本地存储（如硬盘或SD卡）加载未压缩或LZ4压缩的AssetBundle。

在桌面平台，控制台和移动平台上，此API只会加载AssetBundle的头部，并将剩余的数据保留在磁盘上。AssetBundle的 Object  将在以下情况按需加载，调用了加载方法（例如AssetBundle.Load）或 Object 的InstanceID 被引用。在这种情况下不会消耗额外的内存。在Unity编辑器中，API会将整个AssetBundle加载到内存中，就好像从磁盘读取字节并使用 AssetBundle.LoadFromMemoryAsync 一样。如果在Unity编辑器中对项目进行性能分析，此API可能导致在AssetBundle加载期间出现内存峰值。但不影响实际设备性能，在采取补救措施之前应该先在实际设备上对这些峰值进行重新测试。

注意：在Unity 5.3或更早版本的Android设备上，尝试从StreamingAssets路径加载AssetBundles时，此API将失败。 Unity 5.4中已解决该问题。   
在Unity 5.3之前，这个API被称为AssetBundle.CreateFromFile，其功能尚未更改。

#### 3.3.3. AssetBundleDownloadHandler
UnityWebRequest API 允许开发人员准确指定Unity如何处理下载数据，并消除不必要的内存消耗。使用UnityWebRequest下载AssetBundle的最简单方法是调用 UnityWebRequestAssetBundle.GetAssetBundle。

在此我们重点关注 DownloadHandlerAssetBundle。这个类使用工作线程，它会将下载的数据 流式传输到固定大小的缓冲区，然后根据 Download Handler 的配置将缓冲的数据缓冲到 临时存储 或 AssetBundle缓存。所有这些操作都发生在本地代码中，消除了扩展托管堆的风险。此外，Download Handler 不会保留下载字节的 native code 副本，从而进一步减少下载AssetBundle的内存开销。

LZMA压缩的AssetBundles将在下载过程中进行解压缩并使用LZ4压缩进行缓存。通过设置 Caching.CompressionEnabled 可以更改此行为。

下载完成后，Download Handler 的 assetBundle属性 可以获取下载的AssetBundle，就像已经对 下载了的AssetBundle调用了 AssetBundle.LoadFromFile 一样。

如果将缓存信息提供给 UnityWebRequest 对象，并且所请求的 AssetBundle 已经存在于Unity的缓存中，则AssetBundle将立即变为可用，并且此API将以与AssetBundle.LoadFromFile 相同的方式运行。

在Unity 5.6之前，UnityWebRequest 系统使用固定的工作线程池和内部作业系统来防止过度的并发下载。线程池的大小不可配置。在Unity 5.6中，这些保护措施已被删除，以适应更多现代硬件，并允许更快地访问HTTP响应代码和头部。

#### 3.3.4. WWW.LoadFromCacheOrDownload
注意：从Unity 2017.1开始，WWW.LoadFromCacheOrDownload只是包装UnityWebRequest。因此，使用Unity 2017.1或更高版本的开发人员应迁移到UnityWebRequest。 WWW.LoadFromCacheOrDownload将在未来版本中弃用。

以下信息适用于Unity 5.6或更早版本。

WWW.LoadFromCacheOrDownload 允许从远程服务器和本地存储加载 Objects。文件可以通过  file:// URL 的方式从本地存储中加载。如果AssetBundle存在于Unity缓存中，则此API的行为与AssetBundle.LoadFromFile完全相同。

如果AssetBundle尚未被缓存，则 WWW.LoadFromCacheOrDownload 将从其源位置读取AssetBundle。如果AssetBundle是压缩的，它将使用工作线程解压缩并写入缓存。否则，它将通过工作线程直接被写入缓存。一旦AssetBundle被缓存，WWW.LoadFromCacheOrDownload 将从已缓存且解压缩的AssetBundle中加载头信息。然后，对加载的 AssetBundle 采取与 AssetBundle.LoadFromFile 相同的行为。此缓存共享于 WWW.LoadFromCacheOrDownload 和 UnityWebRequest 之间。通过其中一个API下载的AssetBundle也可通过另一个API获得。

虽然数据将被解压缩并通过固定大小写入缓存，但 WWW对象 将在本地内存中保留AssetBundle字节的完整副本。 这个额外副本是为了 WWW.bytes 属性。由于在 WWW对象 中缓存AssetBundle字节的内存开销，AssetBundles应该保持很小 - 最多几兆字节。

与UnityWebRequest不同，每次调用此API都会产生一个新的工作线程。因此，在移动设备等内存有限的平台上，每次只能使用此API下载一个AssetBundle，以避免内存高峰。多次调用此API时，请小心创建过多的线程。如果需要下载超过5个AssetBundle，请使用脚本代码创建和管理下载队列，以确保同时运行的AssetBundle下载不要太多。


#### 3.3.5 建议
一般情况下，应尽可能使用 AssetBundle.LoadFromFile。此API 在 速度、磁盘使用和运行时内存使用等方面都是最高效的。

对于必须下载或补丁 AssetBundles 的项目，强烈建议对于使用 Unity 5.3或之后版本的项目使用 UnityWebRequest，对于使用Unity 5.2或更早版本的项目，则强烈建议使用WWW.LoadFromCacheOrDownload。

使用 UnityWebRequest 或 WWW.LoadFromCacheOrDownload 时，请确保下载器代码在加载AssetBundle后正确调用 Dispose。或者，使用C＃的 using 表达式，以确保便捷的处置 WWW 或UnityWebRequest。

对于需要 独特缓存或下载 的大型工程，可以考虑使用自定义下载器。任何自定义下载程序都应与 AssetBundle.LoadFromFile 兼容。

### 3.4 从AssetBundle中加载资产
从AB中加载 Object 可以使用三个不同的API：   

+ LoadAsset (LoadAssetAsync)    
+ LoadAllAssets (LoadAllAssetsAsync)    
+ LoadAssetWithSubAssets (LoadAssetWithSubAssetsAsync)    

这些API的同步版本总是比异步版本快，至少快一帧。  
异步加载将每帧加载多个对象，取决于它们的时间片限制。  

加载多个独立的 UnityEngine.Objects 时应该使用 LoadAllAssets。最好只在需要加载AssetBundle中的大部分或全部对象时才能使用它。相比于其他两个API，LoadAllAssets比多次单独调用 LoadAssets 稍快。因此，如果要加载的Asset数很大，但需要一次性加载的AssetBundle的比例不到66%，请考虑将AssetBundle拆分为多个较小的包并使用 LoadAllAssets。

LoadAssetWithSubAssets 应该用于加载复合资源，其中包含多个嵌入的 Objects，如 嵌入了动画的 FBX模型，或包括多个精灵的图集。如果需要加载的对象全部来自同一个资产，但与许多其他不相关的对象一起存储在AssetBundle中，则使用此API。

对于任何其他情况，请使用LoadAsset或LoadAssetAsync。

#### 3.4.1 底层加载细节
UnityEngine.Object 加载是在主线程之外执行的：从存储中读取 Object数据 是在工作线程上。 任何不涉及Unity系统的线程敏感部分（脚本，图形）的内容，都在工作线程上进行转换。 例如，从网格创建VBOs，纹理解压缩等。

从Unity 5.3开始，对象加载已经并行化。多个对象在工作线程上被反序列化，处理和集成。 当一个对象完成加载时，它的Awake回调将被调用，并且该对象将在下一帧期间可用于Unity Engine的其余部分。

同步的 AssetBundle.Load 方法将暂停主线程，直到对象加载完成。 它们还会对对象加载进行时间片分割，以便对象集成不会占用超过一定数量的毫秒帧时间。毫秒数由属性Application.backgroundLoadingPriority 设置：   

+ ThreadPriority.High: 每帧最多50ms   
+ ThreadPriority.Normal: 每帧最多10ms   
+ ThreadPriority.BelowNormal: 每帧最多4ms   
+ ThreadPriority.Low: 每帧最多2ms   

从Unity 5.2开始，多个对象被加载，直到达到对象加载的帧时间限制。 假设所有其他因素相同，由于发出异步调用和引擎可用对象之间的最小一帧延迟，资产加载API的异步变体的完成时间总是比对应的同步版本要长。

#### 3.4.2 AssetBundle依赖
AssetBundles 之间的依赖关系使用两个不同的API自动跟踪，具体取决于运行时环境。在Unity编辑器中，可以通过 AssetDatabase API 查询 AssetBundle 依赖关系。AssetBundle的分配和依赖关系，可以通过 AssetImporter API 访问和更改。在运行时，Unity提供了一个可选的API，通过基于 ScriptableObject 的 [AssetBundleManifest](docs.unity3d.com/ScriptReference/AssetBundleManifest.html) API，来加载 AssetBundle 构建期间生成的依赖信息。

当一个或多个父AssetBundle的UnityEngin.Objects引用另一个或多个AssetBundle的UnityEngin.Objects时，AssetBundle依赖于另一个AssetBundle。上面已经介绍了内部引用的一些内容。

如 1.5 序列化和实例化 所述，AssetBundles 充当源数据的作用，此源数据是AssetBundle中每个对象的 FileGUID＆LocalID 标识的源数据的源数据。

因为一个对象在其 Instance ID 被第一次引用时被加载，并且因为一个对象在其AssetBundle 被加载时 被分配了有效的 Instance ID，所以 AssetBundles 加载的顺序并不重要。相反，在加载Object之前，应该先加载包含 该Object所有依赖的 所有AssetBundles。加载父级AssetBundle时，Unity不会尝试自动加载任何子AssetBundles。

例子：
假设 材质A 引用了 贴图B。材质A 被打包到 AssetBundle 1 中，贴图B 被打包到 AssetBundle 2 中。

![]({{ site.url }}/images/post/AssetBundle/AB7.jpg)

加载AB2 必须先于 从 AB1 中加载 Mat A。
这不表示 AB2 必须先于 AB1 加载，或者 Texture B 必须从 AB2 中显示加载。优先加载 AB2 对于从 AB1 中加载 Mat A 已经足够了。

但是，在加载 AB1 时，Unity并不会自动加载 AB2。需要在脚本中手动完成。
对此，前一篇文章 已经介绍过。

#### 3.4.3 AssetBundle manifests
当使用 BuildPipeline.BuildAssetBundles API 执行 AssetBundle 构建管线时，Unity会序列化一个对象，其中包含每个A ssetBundle的依赖信息。此数据存储在一个独立的 AssetBundle 中，其中包含一个单独的 AssetBundleManifest 对象。

此资源将存储在一个 与存放AssetBundles父目录同名的  AssetBundle 中。该AssetBundle可以像其他任何AssetBundle一样加载，缓存和卸载。

AssetBundleManifest 对象提供 GetAllAssetBundles API 来列出与清单并发构建的所有AssetBundles，以及两个方法来查询特定AssetBundle的依赖关系：   

+ [AssetBundleManifest.GetAllDependencies](http://docs.unity3d.com/ScriptReference/AssetBundleManifest.GetAllDependencies.html) 返回需要查询的AssetBundle的所有分层依赖项，其中包括AssetBundle的直接子项，子项的子项等。    
+ [AssetBundleManifest.GetDirectDependencies](http://docs.unity3d.com/ScriptReference/AssetBundleManifest.GetDirectDependencies.html) 只返回AssetBundle的直接子项。     

请注意，这两个API都会分配字符串数组。相应地，它们只应该谨慎使用，而不应该在应用程序生命周期的性能敏感部分中使用。   

#### 3.4.4 建议

在大多情况下，在玩家进入程序的性能关键区（如主场景或世界）之前，最好加载尽可能多的所需对象。 这在移动平台上尤其重要，因为访问本地存储的速度很慢，而且在运行时加载和卸载对象的内存流失可能会触发GC。    
<br/>


## 4 AssetBundle使用模式

### 4.1 管理已加载的资产
在内存敏感的环境下小心控制加载对象的大小和数量非常重要。当对象从当前激活的场景中移除时，Unity不会自动将其卸载。资产清理只在特定时间触发，当然也可以手动触发。

AssetBundles本身必须小心管理。由本地存储上的文件（在Unity缓存中或通过AssetBundle.LoadFromFile 加载）支持的 AssetBundle 具有最小的内存开销（这里应该是说，如果AB来自本地，加载AB只会加载头文件，相较于从网络上下载的AB），很少超过几万字节。但是，如果存在大量的 AssetBundles，则此项开销仍是一个大问题。

由于大多数项目允许用户重新体验内容（例如重新开始），因此知道何时加载或卸载AssetBundle非常重要。<b>如果AssetBundle卸载不当，可能会导致内存中的对象重复。</b>在某些情况下，不当卸载 AB 还会导致不可测行为，例如导致纹理丢失。

管理 资产 和 AssetBundles 时最要了解的是，调用 AssetBundle.Unload 时，unloadAllLoadedObjects 参数设置为 true 和 false 的差异。

该API会卸载已加载的AssetBundle的头信息。 unloadAllLoadedObjects 参数决定是否也卸载从此AssetBundle实例化的所有 Objects。如果设置为true，那么从该AssetBundle实例化的所有对象也将立即被卸载 - 即使它们当前正在活动场景中使用。

例如，假设 材质M 是从 AssetBundle AB 加载的，并且 假设M 当前处于活动场景中。

![]({{ site.url }}/images/post/AssetBundle/AB8.png)

如果调用了 AssetBundle.Unload(true)，那么M将从场景中移除，销毁并卸载。 但是，如果调用 AssetBundle.Unload(false)，则AB的头信息将被卸载，但M仍将保留在场景中并且仍然有效。调用 AssetBundle.Unload(false) 会中断M和AB之间的链接。 如果稍后再次加载AB，则AB中对象的新副本将被加载到内存中。

![]({{ site.url }}/images/post/AssetBundle/AB9.png)

如果稍后再次加载AB，则AssetBundle头信息的新副本将被加载。但是，M并未从AB的这个新副本中加载。Unity不会在 AB的新副本 和 M 之间建立任何关联。

![]({{ site.url }}/images/post/AssetBundle/AB10.png)

如果调用 AssetBundle.LoadAsset() 来重新加载M，则Unity不会将M的旧副本解释为AB中数据的一个实例。 因此，Unity将加载M的新副本，并且在场景中将有两个相同的M副本。

![]({{ site.url }}/images/post/AssetBundle/AB11.png)

对于大多数项目来说，这种行为是不可取的。大多数项目应该使用 AssetBundle.Unload(true) 并采用一种方法来确保对象不重复。两种常用方法是：   

+ 在应用生命周期中有 特定时间点（如在关卡切换或转场读条）卸载 AssetBundles。这是更简单和最常见的选择。      
+ 为每个对象维护一个引用计数，仅在所有实例对象都不使用的时候，卸载AssetBundle。这允许程序卸载并重新加载单个对象而不重复消耗内存。  

如果应用程序必须使用 AssetBundle.Unload(false)，那么单个对象只能通过两种方式卸载：    

+ 对不需要的对象，在场景和代码中同时消除对其所有引用。然后，调用Resources.UnloadUnusedAssets。    
+ 非叠加地加载场景。这将销毁当前场景中的所有对象并自动调用Resources.UnloadUnusedAssets。   

如果一个项目有明确定义的时间点，使用户可以等待对象加载和卸载，那么在这些时间点应该尽量多地 卸载对象 并 加载新的对象。

最简单的方法是将项目的离散块打包到场景中，然后将这些场景及其所有依赖项构建到AssetBundles中。然后，程序可以进入 “Loading” 场景，完全卸载包含旧场景的AssetBundle，然后加载包含新场景的AssetBundle。

虽然这是最简单的流程，但有些项目需要更复杂的AssetBundle管理。由于每个项目都不同，因此没有通用的AssetBundle设计模式。

在决定如何将 Objects 分组到 AssetBundles 时，如果一些 Objects 必须同时加载或更新，通常最好先将它们捆绑到AssetBundles中。例如，一个角色扮演游戏，每个地图和过场动画可按场景分组为AssetBundles，但在大多数场景中都需要一些对象。AssetBundles 可以被构建来提供肖像，UI 以及 角色模型和纹理。后者这些对象和资产可以被分组到第二组 AssetBundles 中，这些 AssetBundles 在启动时加载并在应用的整个生命周期内保持加载状态。

如果Unity必须在一个AssetBundle卸载后从其中重新加载 Object，则可能会出现另一个问题。在这种情况下，重新加载将失败，对象将作为（missing）对象出现在Unity编辑器中。

这主要发生在Unity丢失并恢复对其图形上下文的控制时，例如，当移动应用程序被暂停或用户锁定其PC时。在这种情况下，Unity必须将纹理和着色器重新上传到GPU。如果这些资产的源AssetBundle不可用，则应用程序将以洋红色呈现场景中的对象。


### 4.2 分发
有两种基本方法把一个项目的 AssetBundle 分发给客户端：与项目同时安装或在安装后下载。选择哪种方式主要取决于平台。移动项目通常选择后一种，主机和PC项目通常选择前一种。    
正确的程序框架允许在安装后将新内容或修补的内容作为补丁更新，而不用关心初始的AssetBundles。

#### 4.2.1 跟随项目发布
将AssetBundles与项目一起发布是最简单的发布方式，因为它不需要额外的下载管理代码。 跟随项目发布主要有两个原因：    

+ 减少项目构建时间并允许更简单的迭代开发。 如果这些AssetBundles不需要与应用程序本身分开更新，那么AssetBundles可以存储在 StreamingAsset 文件夹中从而包含在应用中。   
+ 发布可更新内容的初始修正版。通常这样做是为了节省终端用户在初次安装后的时间，或者作为以后打补丁的基础。 SteamingAsset 对于这种情况并不是最理想的方案。 但是，如果不愿意编写一个自定义下载和缓存的系统，那么可以使用这种方法，从StreamingAssets 将可更新内容的初始修正版加载到Unity缓存中。  

#### 4.2.1.1 StreamingAsset
让应用在安装时 包含任何类型的内容（包括AssetBundles），最简单方法是在构建项目之前将内容构建到 /Assets/StreamingAssets 文件夹中。构建时包含在StreamingAssets文件夹中的任何内容都将被复制到最终的应用程序中。

通过属性 Application.streamingAssetsPath 可以获得本地存储上的 StreamingAssets 文件夹完整路径。然后在大多数平台上，可通过 AssetBundle.LoadFromFile 加载AssetBundles。

<b>Android开发人员</b>：在Android上，StreamingAssets文件夹中的资源文件会存储到APK中，并且如果它们是压缩的，可能需要更多时间加载，因为存储在APK中的文件可能使用不同的存储算法。使用的算法可能会因Unity版本而异。您可以使用7-zip等解压工具打开APK以确定这些文件是否被压缩。如果被压缩，AssetBundle.LoadFromFile() 将执行得更慢。在这种情况下，作为变通方案您可以使用 UnityWebRequest.GetAssetBundle 来检索已经缓存的版本。通过使用UnityWebRequest，AssetBundle将在第一次运行时解压缩并缓存，从而使后续执行速度更快。请注意，这将需要更多的存储空间，因为AssetBundle将被复制到缓存中。或者，您可以导出您的Gradle项目，并在构建时向您的AssetBundles添加扩展名。然后，您可以编辑 build.gradle 文件并将该扩展名添加到noCompress 部分。完成后，您应该可以使用AssetBundle.LoadFromFile() 而无需消耗解压缩的性能成本。

注意：StreamingAsset在某些平台上不是可写文件。如果安装后需要更新项目的AssetBundles，则可以使用 WWW.LoadFromCacheOrDownload 或编写自定义下载程序。

#### 4.2.2 下载后安装
将AssetBundles部署到移动设备的最佳方法是在安装后下载。这也允许不强制用户更新应用而在安装后更新内容。许多平台上，应用二进制文件必须经过昂贵且冗长的复审过程。因此，开发一个良好的安装后下载系统至关重要。

交付AssetBundles的最简单方法是将它们放置在Web服务器上并通过 UnityWebRequest 部署。Unity会自动将下载的 AssetBundles 缓存在本地存储上。如果下载的AssetBundle是LZMA压缩的，则AssetBundle将以未压缩或重新压缩为LZ4（取决于Caching.compressionEnabled设置）的形式存储在缓存中，以便将来加载更快。如果下载的捆绑包压缩了LZ4，则AssetBundle将被压缩存储。如果缓存填满，Unity将从缓存中删除最近最少使用的AssetBundle。

通常建议尽可能使用 UnityWebRequest，或者仅在使用Unity 5.2或更早版本时使用 WWW.LoadFromCacheOrDownload。对于特定项目，如果内置API的内存消耗、缓存行为或性能影响很大，或者项目必须运行于特定平台，那么只能自定制下载系统了。

使用 UnityWebRequest 或 WWW.LoadFromCacheOrDownload 可能不理想的情况示例：    

+ 当需要对AssetBundle缓存进行细粒度控制时    
+ 当项目需要实施自定义压缩策略时     
+ 当项目希望使用平台特定的API来满足某些要求时，例如需要在后台传输数据。   
    + 示例：使用iOS的 Background Tasks API 在后台下载数据。   
+  如果AssetBundles必须通过SSL下载，但是Unity没有正确的SSL支持（如PC）。

#### 4.2.3 内置缓存
Unity有一个内置的AssetBundle缓存系统，可用于缓存通过 UnityWebRequest API 下载的 AssetBundles，该API包含一个接受AssetBundle版本号作为参数的重载。此版本号不存储在AssetBundle内部，并且不是由AssetBundle系统生成。

缓存系统持续跟踪传递给 UnityWebRequest 的最新版本号。当使用一个版本号调用此API，缓存系统通过比较版本号来检查是否存在已缓存的AssetBundle。如果这些数字匹配，系统将加载缓存的AssetBundle。如果版本号不匹配，或没有缓存的AssetBundle，Unity将下载一个新副本。这个新副本将与新版本号相关联。

<b>缓存系统中的 AssetBundles 仅由其文件名来标识</b>，而不是由其下载的完整URL标识。这意味着 一个AssetBundle的同名文件 可以存储在多个不同的位置，例如CDN。只要文件名称相同，缓存系统就会将它们识别为相同的AssetBundle。

每个应用确定 将版本号分配给 AssetBundles 的适当策略，并将这些版本号传递给UnityWebRequest。这些数字可能来自各种唯一标识符，例如CRC值。请注意，虽然AssetBundleManifest.GetAssetBundleHash() 也可用于此目的，但我们不建议使用此功能进行版本控制，因为它仅提供估算值，而不是真正的Hash值计算。

在Unity 2017.1以后，缓存API已经扩展到提供更精细的控制，允许开发人员从多个缓存中选择一个活动缓存。以前的Unity版本只能修改 Caching.expirationDelay 和Caching.maximumAvailableDiskSpace 来删除缓存的资源（Unity 2017.1中这些属性保留在 Cache 类中）。

expirationDelay 是自动删除AssetBundle之前必须经过的最小秒数。如果在此期间没有访问AssetBundle，它将被自动删除。

maximumAvailableDiskSpace 指定本地存储空间量（以字节为单位），这个量是指删除已超过 expirationDelay时间的AssetBundle之前可以使用的空间量。达到限制时，Unity将删除最近最少打开的缓存中的AssetBundle（或通过 Caching.MarkAsUsed 标记为已使用）。 Unity会删除缓存的AssetBundles，直到有足够的空间完成新的下载为止。

#### 4.2.3.1 缓存填充
由于AssetBundles使用文件名作为标识，所以可以使随用程序附带的AssetBundles“准备”缓存。 为此，请将每个AssetBundle的初始或基本版本存储在 /Assets/StreamingAssets/ 中。  

第一次运行应用程序时，可以通过从Application.streamingAssetsPath 加载AssetBundles 来填充缓存。 此后，应用程序可以正常调用UnityWebRequest（UnityWebRequest也可用于最初从StreamingAssets路径加载AssetBundles）。


#### 4.2.4 自定义下载器
编写自定义下载程序可以让应用程序完全控制 AssetBundles 的下载、解压缩和存储方式。由于所涉及的工程量是很复杂的，所以只推荐大型团队使用此方法。编写自定义下载器时有四个主要考虑事项：    

+ 下载机制    
+ 存储位置    
+ 压缩类型    
+ 补丁   

#### 4.2.4.1 下载  
对于大多数应用程序，HTTP是下载AssetBundles最简单的方法。然而，实现基于HTTP的下载程序并不简单。自定义下载程序必须避免 过多的内存分配、过多的线程使用 和 过多的线程唤醒。Unity的 WWW 类不适合的原因在上面 3.3.4 已介绍。

在编写自定义下载器时，有三个选择：   

+ C＃的 HttpWebRequest 和 WebClient 类    
+ 自定义原生插件     
+ Asset Store 包  

#### 4.2.4.1.1 c#类
如果应用程序不需要使用 HTTPS/SSL，那么C＃的 WebClient 类提供了下载AssetBundles最简单的机制。它能够将任何文件直接异步下载到本地存储，而无需过多管理内存分配。

要使用 WebClient 下载AssetBundle，请分配该类的一个实例，并将 AssetBundle的下载URL 和 目标路径 传递给它。如果需要对请求参数进行更多控制，可以使用C＃的 HttpWebRequest 类编写下载程序：    

+ 从 HttpWebResponse.GetResponseStream 获取一个字节流。    
+ 在堆上分配一个固定大小的字节缓冲区。   
+ 从响应流（reponse）中读入缓冲区。    
+ 使用C＃的 File.IO API 或任何其他流式IO系统将缓冲区写入磁盘。   

#### 4.2.4.1.2 Asset Store包
很多插件包提供了原生代码的实现，以通过HTTP，HTTPS和其他协议下载文件。 在为Unity编写自定义本机代码插件之前，建议您先查找可用的Asset Store包。

#### 4.2.4.1.3 自定义原生插件
编写自定义原生插件是在Unity中下载数据最耗时，但最灵活的方法。由于编程时间花费高且技术风险高，只有在没有其他方法能够满足应用程序的要求时才推荐此方法。例如，如果应用程序必须在Unity中没有C＃SSL支持的平台上使用SSL通信，则可能需要定制原生插件。

自定义原生插件通常会封装目标平台的原生下载API。比如，iOS上的 NSURLConnection 和 Android上的 
java.net.HttpURLConnection。请查阅每个平台的文档以获取有关使用这些API的详细信息。

#### 4.2.4.2 存储
在所有平台上，Application.persistentDataPath 指向一个可写的位置，适合用来存储应用运行时要保持的数据。在编写自定义下载器时，强烈建议使用 Application.persistentDataPath 的子目录来存储下载的数据。

Application.streamingAssetPath 不可写，对于AssetBundle缓存来说是一个糟糕的选择。 streamingAssetsPath的位置包括：    

+ OSX：在.app包内; 不可写   
+ Windows：在安装目录中（例如Program Files）; 通常不可写     
+ iOS：在.ipa包内; 不可写     
+ Android：在.apk文件中; 不可写   


### 4.3 资产分配策略
如何将项目资产划分为AssetBundles并不简单。经常采用简单的策略，比如将所有对象都各自生成一个AssetBundle 或 仅使用一个AssetBundle，但这些解决方案具有明显的缺点。相关内容前一篇文章已经介绍过。

### 4.4 常见陷阱
本节介绍使用AssetBundles时常见的几个问题。

#### 4.4.1 资产重复
Unity 5的AssetBundle系统，当Objects构建到AssetBundle中时会查找对象的所有依赖关系。此依赖项信息用于确定将包含在AssetBundle中的对象集。

显式分配给AssetBundle的对象将仅构建到该AssetBundle中。当 Object 的 AssetImporter 的 assetBundleName属性 设置为非空字符串时，对象将被“显式分配”。这可以通过在 Inspector中选择一个AssetBundle进行设置 或 从脚本中完成。

也可以将对象定义为 AssetBundle building map 的一部分 来将其分配给AssetBundle，方法是，该map要与重载的 BuildPipeline.BuildAssetBundles() 函数一起使用，该函数接受一组AssetBundleBuild。

任何未在AssetBundle中显式分配的对象，若有 一个或多个对象 引用此未标记对象，该对象都将包含在所有含有这些引用它的对象的AssetBundles中。

例如，如果将两个不同的对象分配给两个不同的AssetBundles，但都具有对公共依赖项Object的引用，则该依赖Object将被复制到两个AssetBundles中。重复的依赖关系也将被实例化，这意味着依赖对象的两个副本将被视为具有不同标识符的不同对象。这将增加应用程序AssetBundles的总大小。如果应用程序加载它的父项，这也会导致Object的两个不同副本被加载到内存中。

有几种方法可以解决这个问题：    

+ 确保构建到不同AssetBundles中的对象不共享依赖关系。任何共享依赖关系的对象都可以放置到同一AssetBundle中，而无需复制它们的依赖项。    
    + 对于具有许多共享依赖的项目，此方法通常不可行。为了方便和高效的使用，它将导致频繁地重建和重新下载 一个庞大的 AssetBundles。    

+  分段AssetBundles，以便不会同时加载共享依赖项的两个AssetBundles。   
    + 此方法可能适用于某些类型的项目，例如基于关卡的游戏。但是，它仍然会不必要地增加项目的AssetBundles的大小，并增加构建时间和加载时间。    
   
+ 确保所有依赖项资产都生成一个它们自己的AssetBundles中。这完全消除了重复资产的风险，但也带来了复杂性。应用必须在AssetBundles之间跟踪依赖关系，并确保在调用任何 AssetBundle.LoadAsset API 之前加载了正确的AssetBundles。

对象依赖关系是 通过 UnityEditor名称空间 中的 AssetDatabase API 来跟踪。正如命名空间所示，这个API仅在Unity编辑器中可用，而不是在运行时。 AssetDatabase.GetDependencies 可用于查找特定对象或资产的所有直接依赖关系。请注意，这些依赖关系可能有其自己的依赖关系。此外，AssetImporter API 可用于查询分配有任何指定对象的AssetBundle。

通过组合 AssetDatabase 和 AssetImporter API，可以编写一个编辑器脚本，以确保将所有 AssetBundle 的直接或间接依赖关系都生成AssetBundle，或者没有两个 AssetBundles 共享尚未分配给AssetBundle的依赖项。由于重复资产会产生的内存成本，建议所有项目都有这样的脚本。


#### 4.4.2 图集重复
任何自动生成的图集将被分配给一个AssetBundle，该AB包含生成自图集的Sprite对象。如果Sprite对象被分配给多个AssetBundles，那么图集将不会只分配给一个AssetBundle，而是多个。如果Sprite对象未分配给AssetBundle，则图集也不会被分配给AssetBundle。

为了确保图集没有重复，请检查在同一图集中的所有Sprite都分配到同一个AssetBundle。

请注意，在 Unity 5.2.2p3 及更早版本中，自动生成的图集将永远不会分配给AssetBundle。因此，它们将被包含在使用了该Sprite的任何AssetBundles中，以及任何引用其组成Sprite的AssetBundles中。由于这个问题，强烈建议所有使用Unity的sprite打包程序的Unity 5项目升级到Unity 5.2.2p4,5.3或任何更新版本的Unity。

#### 4.4.3 Android纹理
由于Android环境的设备碎片化严重，通常需要将纹理压缩为多种不同的格式。虽然所有Android设备都支持ETC1，但ETC1不支持带alpha通道的纹理。如果应用程序不需要OpenGL ES 2支持，则最简单解决方法是使用所有 Android OpenGL ES 3 设备支持的ETC2。

大多数应用程序需要在不支持ETC2的旧设备上发布。解决此问题的一种方法是使用 Unity 5 的 AssetBundle变体（有关其他选项的详细信息，请参阅Unity的Android优化指南）。

要使用AssetBundle变体，所有无法使用ETC1进行压缩的纹理必须分离为仅包含纹理的AssetBundles。接下来，使用特定于供应商的纹理压缩格式（如DXT5，PVRTC和ATITC）创建相应的这些AssetBundles变体以支持Android中不支持ETC2的设备。对于每个AssetBundle变体，将包含的纹理的 TextureImporter 设置更改为适合Variant的压缩格式。

在运行时，可以使用 SystemInfo.SupportsTextureFormat API 检测对不同纹理压缩格式的支持。并使用此信息来选择和加载包含以受支持格式压缩的纹理的AssetBundle变体。

有关Android纹理压缩格式的更多信息可以在 [这里](developer.android.com/guide/topics/graphics/opengl.html#textures) 找到。


#### 4.4.4 iOS文件句柄过度使用

当前版本Unity不受此问题影响。

在Unity 5.3.2p2之前的版本中，Unity会在AssetBundle加载的整个过程中持有一个打开的文件句柄。这在大多数平台上都不是问题。 但是，iOS将进程可以同时打开的文件句柄数限制为255，如果加载AssetBundle会导致超出限制，则加载调用将失败，并显示 “Too Many Open File Handles” 错误。

对于尝试将内容分成数百或数千个AssetBundles的项目，这是一个常见问题。

对于无法升级到补丁版本的Unity的项目，临时解决方案是：   

+ 通过合并相关的AssetBundles来减少使用的AssetBundles的数量。
+ 使用AssetBundle.Unload(false) 关闭AssetBundle的文件句柄，并手动管理加载的对象的生命周期。   


### 4.5 AssetBundle 变体(Variants)
AssetBundle系统的一个关键特性是引入了AssetBundle变体。变体的目的是允许应用调整其内容以更好地适应其运行时环境。当 加载对象 和 解析 Instance ID 引用时，变体允许 不同的 AssetBundle文件中的 不同 UnityEngine.Objects 显示为 “相同” 的对象。概念上讲，它允许两个 UnityEngine.Objects 显示为共享同样的 File GUID & Local ID，并标识实际的 UnityEngine.Object 以字符串变体ID加载。

这个系统有两个主要用例：    

+ 变体简化了适用于指定平台的AssetBundles加载。    
    + 示例：构建系统可能会创建一个AssetBundle，其中包含适用于 DirectX11 Windows 版本的高分辨率纹理和复杂着色器，以及适用于 Android 的具有较低保真度内容的第二个AssetBundle。在运行时，项目的资源加载代码可以为其平台加载相应的AssetBundle Variant，并且传递到 AssetBundle.Load API 的对象名称不需要更改。

+ 变体允许应用程序在同一平台但不同硬件上，加载不同的内容。    
    + 这是支持各种移动设备的关键。在现实的应用程序中，iPhone 4 不能像最新的iPhone一样显示相同的清晰度。    
    + 在Android上，AssetBundle 变体可用于解决设备间屏幕纵横比和DPI之间巨大的分裂问题。


#### 4.5.1 变体 限制
AssetBundle 变体系统的一个关键限制是它需要使用不同的资产来构建变体。即使这些资产之间的唯一差异是其导入设置，也适用此限制。如果内置到 变体A 和 变体B 中的纹理之间，唯一区别就是在Unity纹理导入器中选择的特定纹理压缩算法，变体A 和 变体B 依然是完全不同的资产。这意味着变体A和变体B是磁盘上的两个文件。   

这种限制使大型项目的管理复杂化，因为特定资产的多个副本必须保存在源代码管理中。当开发人员希望更改资产的内容时，必须更新资产的所有变体。这个问题没有内置的解决方法。   

大多数团队都有他们自己的AssetBundle变体形式。这是通过使用明确定义的文件后缀名 来构建AssetBundles 完成的，以便识别给定AssetBundle所代表的特定变体。在构建这些AssetBundles时，通过代码更改包含的资产的导入器设置。一些开发者已经扩展了他们的定制系统，以便能够改变 Prefab 上组件 的参数。


### 4.6 压缩还是不压缩？
是否压缩 AssetBundles 需要一些重要的考虑因素，其中包括：    

+ 加载时间：从本地存储或本地缓存加载时，未压缩的AssetBundles比加载压缩的AssetBundles要快得多。     
+ 构建时间：在压缩文件时，LZMA和LZ4非常缓慢，Unity Editor按顺序处理AssetBundles。具有大量AssetBundles的项目将花费大量的时间压缩它们。     
+ 应用程序大小：如果AssetBundles包含在应用程序中发布，则压缩它们将减少应用程序的总大小。或者，可以在安装后下载AssetBundles。     
+ 内存使用情况：在Unity 5.3之前，所有Unity的解压缩机制都要求在解压缩之前将整个压缩的AssetBundle加载到内存中。如果内存使用率很重要，请使用未压缩或LZ4压缩的AssetBundles。     
+ 下载时间：如果AssetBundles很大，或者用户处于带宽受限的环境中，例如在低速或流量上下载，那么可能需要压缩。如果只有几兆字节的数据通过高速连接传送到PC，则可能会忽略压缩。     

#### 4.6.1 Crunch压缩
如果Bundles中，主要由使用Crunch压缩算法的DXT压缩纹理组成，则应该被构建为未压缩。   


### 4.7 AssetBundles和WebGL 
由于Unity的WebGL导出选项当前不支持辅助线程，因此WebGL项目中的所有AssetBundle解压缩和加载都必须在主线程上进行。AssetBundles的下载委托给浏览器使用 XMLHttpRequest，XMLHttpRequest在Unity的主线程上执行。一旦下载完，压缩的AssetBundles将在Unity的主线程上解压，因此根据包的大小将不同程度地延迟Unity内容的执行。

Unity建议最好使用更小的 AssetBundle，以避免出现性能问题。与使用大 AssetBundle 相比，此方法还具有更高的内存效率。Unity WebGL只支持LZ4压缩和未压缩的 AssetBundle，但是，可以对Unity生成的 bundles 使用 gzip/brotli 压缩。在这种情况下，您需要相应地配置Web服务器，以便文件在下载时被浏览器解压。有关更多详细信息，请参见 here。

如果您使用的是Unity 5.5或更高版本，请考虑避免使用LZMA压缩您的AssetBundles，而使用LZ4。Unity 5.6删除了LZMA作为WebGL平台的压缩选项。

