---
layout: post
title: AssetBundle 基础总结
description: "AssetBundle 基础总结"
modified: 2019-01-27
tags: [Unity]
---

虽然之前已经写过AssetBundle的内容了，不过都比较久了，最近打算再把相关内容总结一下，主要都是Unity官方文档和Best Practice的内容。

### 介绍
1. AssetBundle是一个压缩包，包含模型、材质、Prefab、音效、整个场景，可以在游戏运行的时候载入。
2. AssetBundle有互相依赖的关系。
3. AssetBundle可以使用LZMA、LZ4的压缩演算法，减少资源包大小，增加网路传输的速度。
4. AssetBundles用于可下载内容（DLC），减少安装包大小，针对行动平台下载优化后的资源，可减少运行时的内存压力。  
<br/>  

### AssetBundle是什么
1. 它是存在于硬盘上的文档，可以称为一个压缩包。压缩包内包含有资料夹、各种档案，主要分为两种类：Serialized File、Resource File。
2. Serialized File：可序列化文档，资源会被打碎后统一放入一个单一文档中。
3. Resource File：资源文档，比如声音、图片材质会被单独保存，方便快速加载。
4. 我们可以通过代码从特定的AssetBundle中，加载所有我们当初加入到AssetBundle中的内容。 
   
![]({{ site.url }}/images/post/AssetBundle/AB4.png)  
普通 Asset Bundle 结构   

![]({{ site.url }}/images/post/AssetBundle/AB5.png)
Streamed Scene Asset Bundle 结构   
<br/>  

### AssetBundle 工作流
1. 指定资源到AssetBundles
2. 创建AssetBundles
3. 上传AssetBundles到其他服务器
4. 加载AssetBundles与内部资源  
<br/>  

### 指定资源
选择需要打包的资源，右下角指定 AssetBundle name 和 Variant name（可选）。AssetBundle names支持文件夹格式，使用 / 分隔。
也可以使用脚本指定：  

{% highlight c# %}  
[MenuItem("Tools/BuildAssetbundle")]
static void BuildAssetbundle(){
    string outPath = Path.Combine(Application.dataPath, "StreamingAssets");
    //如果目录已经存在删除它
    if (Directory.Exists (outPath)) {
        Directory.Delete (outPath,true);
    }
    Directory.CreateDirectory (outPath);
    
    List<AssetBundleBuild> builds = new List<AssetBundleBuild> ();
    //设置bundle名，和多少资源打在同一个Bundle内
    builds.Add (new AssetBundleBuild (){assetBundleName="Cube.unity3d",assetNames =new string[]{"Assets/Cube1.prefab","Assets/Cube2.prefab"}});
    builds.Add (new AssetBundleBuild (){assetBundleName="CubeMatarial.unity3d",assetNames =new string[]{"Assets/CubeMatarial.mat"}});
    //构建Assetbundle
    BuildPipeline.BuildAssetBundles (outPath,builds.ToArray(),BuildAssetBundleOptions.ChunkBasedCompression, BuildTarget.StandaloneOSX);
    //刷新
    AssetDatabase.Refresh ();
}
{% endhighlight %}  
<br/>  

### 分组策略
错误的分组策略：
有一种很简单的分组策略方式，比如将资源分别放置在单独的AssetBundle中，或将所有资源都打包于一个AssetBundle中，但这样的分组策略具有明显的缺点：  
拥有太少的AssetBundles：1. 增加运行时内存使用量。2. 增加加载时间。3. 需要更大的下载量。   
拥有太多的AssetBundles：1. 增加构建时间。2. 可能会使开发复杂化。3. 增加总下载时间。   

好的分组策略：  
1. 逻辑实体分组    
2. 资源类型分组    
3. 并发内容分组   

#### 逻辑实体分组
1. 对于一个UI界面，其包含的所有贴图与布局数据。
2. 对于一个角色，其包含的角色模型与动画。
3. 多个场景所共享的场景碎片（材质与模型）资源打一个包。   
   
逻辑实体分组对于可下载内容（DLC）非常理想，因为通过这种方式可以分离各个内容，可以对单个实体进行更改，而不需要下载其他未更改的资产。  
能够正确实施这一策略的最大诀窍是，开发人员必须熟悉项目何时何地使用每种资产。

#### 类型分组
按照类型分组，比如所有音频一个包，所有模型一个包，然后材质一个包。  
这是为多平台使用构建AB的较好策略。 例如，如果您的音频压缩在Windows和Mac平台之间相同，则可以将所有音频打包到AssetBundles中。而shader倾向于使用更多特定平台的选项进行编译，因此您为Mac构建的shader包可能会不能在windows上重用。  
这种方法非常适合让您的AssetBundles与更多的游戏版本兼容，毕竟随着游戏版本更新，更多修改的是代码脚本与prefab，而如同纹理压缩格式和设置一类的修改频率低，可都放于同一个AB。  

#### 并发内容分组
这一策略是把同一时间将会加载和使用的资源打包到一起。  
可以按照关卡分类，一个关卡所需要的资源包括角色、贴图、声音打包成一个AssetBundle。也可以按照场景分类，一个场景所需的资源打包在一起。  

#### 策略总结
请注意，一个项目应该根据需要混合上述的三种策略。  
例如，项目可能决定将其用于不同平台的用户界面（UI）元素分组到他们自己的Platform-UI特定包中，但另外按照关卡/场景对其交互式内容进行详细的分组。    
一些应注意的：  
1. 把经常更新的资源放在同一个包，跟不经常更新的包分离。
2. 把需要同时加载的放在同一个包。
3. 把其他多个包里的对象依赖的资源，分离出来放在一个单独的包。
4. 有些资源不可能同时加载，应分离出来，如标准解析度与高解析度的资源。
5. 如果有一个包只有50%的内容经常加载，应该要进行分割。
6. 把小的AssetBundle（少于5到10个资源）合并成同一个包。
7. 如果同一个资源有两个版本，可以考虑使用后缀名来区分。

依赖关系实例：两个prefab都依赖了同一份材质和和贴图文件。如果两个prefab，分别打包，会得到两个AB大小都为500KB（假设）。但是如果将它们的材质和贴图单独打一个包，再把两个prefab分别打包，即三个包，会得到：材质贴图包450KB，两个prefab的包 600 byte。即两个prefab会自动依赖材质贴图。  
<br/>  

### 构建AssetBundles
打包 Editor 中所有指定了的AB：  
public static AssetBundleManifest BuildAssetBundles(string outputPath, BuildAssetBundleOptions assetBundleOptions, BuildTarget targetPlatform);

从给定的 building map 打包：  
public static AssetBundleManifest BuildAssetBundles(string outputPath, AssetBundleBuild[] builds, BuildAssetBundleOptions assetBundleOptions, BuildTarget targetPlatform);

#### BuildAssetBundleOptions
BuildAssetBundleOptions.None：    
使用LZMA算法压缩，压缩的包更小，但是加载时间更长。使用之前需要全部解压缩。  
一旦包被解压缩，它将使用LZ4在磁盘上重新压缩并储存到本地端，所以使用资源的时候不需要全部解压缩。   

BuildAssetBundleOptions.UncompressedAssetBundle:   
不压缩，容量大，但加载时间最快。

BuildAssetBundleOptions.ChunkBasedCompression:   
使用LZ4算法压缩，容量比LZMA压缩后大，但好处是使用资源时不用全部解压缩，只要解压缩所需的『Chunk』部分即可，因此又被称为Chunk-Based演算法。  

如果使用LZ4算法压缩，加载速度与不加缩的方法一样快，但容量比不压缩还小。

#### Manifest 文件
对于生成的每个包（包括附加的Manifest包），都会生成关联的清单文件。 清单文件可以用任何文本编辑器打开，并包含诸如循环冗余校验（CRC）数据和依赖性数据等信息。
Manifest文件是以YAML格式储存，可以从Assets栏位看见该AssetBundle内部储存了哪些资源。Dependencies则会显示是否有依赖其他的AssetBundle。

{% highlight c# %}  
ManifestFileVersion: 0
CRC: 2422268106
Hashes:
  AssetFileHash:
    serializedVersion: 2
    Hash: 8b6db55a2344f068cf8a9be0a662ba15
  TypeTreeHash:
    serializedVersion: 2
    Hash: 37ad974993dbaa77485dd2a0c38f347a
HashAppended: 0
ClassTypes:
- Class: 91
  Script: {instanceID: 0}
Assets:
  Asset_0: Assets/Mecanim/StateMachine.controller
Dependencies: {}
{% endhighlight %}  

+ CRC：循环冗余校验。  
+ AssetFileHash：AssetBundle中所有资产文档的Hash，只用来做增量创建时的检查。  
+ TypeTreeHash：AssetBundle中所有类型的Hash，只用来做增量创建时的检查。  
+ ClassTypes：AssetBundle中包含的所有类型，这些数据用来得到一个新的单独的Hash。当typetree做增量构建检测。  
+ Assets：所有在AssetBundle中的资产的名称。  
+ Dependencies：AssetBundle所依赖的其它AssetBundle的名字。  
  
这个清单文档只是用来做增量构建的，非运行时必须。

Dependencies还有一个作用，就是当AssetBundle因分组策略的关系，将部分『共享的资源』另外打包成一个AssetBundle，这时就需要载入这个有Dependencies的AssetBundle。从AssetBundle.manifest文件可以看见所有AssetBundle与Dependencies。

例子：Bundle1中的Material引用了Bundle2中的Texture：
在从Bundle 1加载Material之前，您需要将Bundle 2加载到内存中。 加载Bundle 1和Bundle 2的顺序无关紧要，重要的是在从Bundle 1加载Material之前加载Bundle 2。  
<br/>  

### 本地使用AssetBundles
根据加载平台和压缩方式的不同，有很多API可以加载AB：   

+ AssetBundle.LoadFromMemoryAsync  
+ AssetBundle.LoadFromFile  
+ WWW.LoadfromCacheOrDownload  
+ UnityWebRequest’s DownloadHandlerAssetBundle (Unity 5.3 or newer)  

#### AssetBundle.LoadFromMemoryAsync
以AB数据的 bytes数组 为参数。  
如果包是LZMA压缩的，它将在加载时对AssetBundle进行解压缩。LZ4压缩包以压缩状态加载。  

从内存加载AssetBundle，比方说从网络上下载的文件是bytes数组，可以直接使用LoadFromMemoryAsync转换成AssetBundle。Unity官方不建议使用此API，原理上至少会占用三倍AssetBundle的容量于内存内。

#### AssetBundle.LoadFromFile
对于从本地加载未压缩AB时很高效。  
如果包是未压缩的或块(LZ4)压缩的，LoadFromFile将直接从磁盘加载包。  
如果包是完全压缩(LZMA)的将首先解压缩包，然后再将其加载到内存中。  
  
从本地加载AssetBundle。如AssetBundle为LZ4压缩格式，则只会加载 AssetBundle 的档头，其余的数据仍留存在硬盘上，将会基于「按需求 (on-demand)」的模式来加载。  

*注意： Android设备上，Unity5.3及之前版本，使用此API 从 Streaming 加载会出错，因为那个路径是压缩的.jar文件。Unity5.4 及之后版本可正常使用。

#### WWW.LoadFromCacheOrDownload （将抛弃）
从远程服务器或本地下载AssetBundle后，在本地缓存并载入。  
如果AB是压缩的，会有一个工作线程进行解压缩并写入缓存。一旦压缩并缓存完，将像调用AssetBundle.LoadFromFile 一样加载。  

*注意：从Unity 2017.1开始，此方法包装自UnityWebRequest。 推荐使用UnityWebRequest。   
与UnityWebRequest不同，每次调用此API都会产生一个新的工作线程。因此，在移动设备等内存有限的平台上，每次只能使用此API下载一个AssetBundle，以避免内存高峰。如果需要下载超过5个AssetBundle，请使用代码创建和管理下载队列，以确保只有少量AssetBundle下载正在同时运行。  

#### UnityWebRequest 
首先，需要使用 UnityWebRequest 创建Web请求。 返回请求后，将请求对象传递给DownloadHandlerAssetBundle.GetContent（UnityWebRequest）。 此GetContent调用将返回您的AssetBundle对象。  
下载AB后，您还可以在DownloadHandlerAssetBundle类上使用assetBundle属性，以使用AssetBundle.LoadFromFile的效率加载AssetBundle。  
使用DownloadHandlerAssetBundle下载时，下载的数据会传输到固定大小的缓冲区，然后根据不同的配置方式，决定将缓冲的数据放到暂存空间或AssetBundle缓存空间。 以上操作都以native-code形式进行，消除了扩展heap的风险。  

{% highlight c# %}  
IEnumerator Start()
{
    string url = "https://website.com/assetbundle";
    using (var uwr = new UnityWebRequest(url, UnityWebRequest.kHttpVerbGET))
    {
        uwr.downloadHandler = new DownloadHandlerAssetBundle(url, 0);
        yield return uwr.SendWebRequest();
        AssetBundle bundle = DownloadHandlerAssetBundle.GetContent(uwr);
    }
}
{% endhighlight %}  

需要注意，通过UnityWebRequest下载的 AB 并不会保存到本地缓存。你要根据需要自己管理下载的AB，可通过 uwr.downloadHandler.data 获得 byte 数组，再使用 File.WriteAllBytes 写入本地。  
<br/>  

### 从 AssetBundle 加载资源
通用代码 

{% highlight c# %}  
// 单个
T objectFromBundle = bundleObject.LoadAsset<T>(assetName);
// 所有
Unity.Object[] objectArray = loadedAssetBundle.LoadAllAssets();
{% endhighlight %} 

异步
{% highlight c# %}  
AssetBundleRequest request = loadedAssetBundleObject.LoadAssetAsync<GameObject>(assetName);
yield return request;
var loadedAsset = request.asset;
{% endhighlight %}   
<br/>  


### 加载 AssetBundle Manifests
加载  Manifests 很有用，尤其对于处理依赖关系。
要获得可用的AssetBundleManifest对象，需要加载附加的AssetBundle（与所在的文件夹同名）并从中加载 AssetBundleManifest类型 的对象。
然后你就可以通过 manifest 获得 AssetBundles 的信息，如依赖内容等。

{% highlight c# %}  
AssetBundle assetBundle = AssetBundle.LoadFromFile(manifestFilePath);
AssetBundleManifest manifest = assetBundle.LoadAsset<AssetBundleManifest>("AssetBundleManifest");
string[] dependencies = manifest.GetAllDependencies("assetBundle"); //Pass the name of the bundle you want the dependencies for.
foreach(string dependency in dependencies)
{
    AssetBundle.LoadFromFile(Path.Combine(assetBundlePath, dependency));
}   
{% endhighlight %}   
<br/>   


### 管理加载的 AB
如果将游戏对象从场景中Destroy，Unity也不会自动卸载AssetBundle，即便你后面都不使用，已加载的AssetBundle仍占用内存。  
除非你更换场景(Scene)，更换的时候Unity会自动Destroy所有场景中的游戏对象，并接着调用Resources.UnloadUnusedAssets，这个函数会将没有用到的Asset从内存中卸载。  

AB管理最重要的就是理解何时调用 AssetBundle.Unload(bool)，以及该传递true还是false。该参数表示 是否还卸载从此AssetBundle实例化的所有对象。

AssetBundle.Unload(true)  会卸载所有从AB加载的 GameObjects (及其依赖)。但是并不包括拷贝的GO，比如 Instantiated 的GO，因为它们已经不属于AB。

对于一个加载于AB的材质M，如果调用 AB.Unload(true) ，当前场景所有M的实例都会被Unload并销毁。如果调用 AB.Unload(false) ，将会断开M实例和AB的连接，当再次加载AB并调用 AB.LoadAsset()后，并不会重新连接M和新的材质，而是会生成两份材质的拷贝。

Unity官方並不建議使用AssetBundle.Unload(false)，因為管理不當的話，有可能導致場景中有多份以上的相同Asset。

大多数项目应该使用AssetBundle.Unload（true）。 两种常用方法是：   

+ 在整个生命周期中有明确使用Unload的时机，例如在关卡之间或在加载期间卸载临时的AssetBundles。  
+ 对每个对象进行引用计数，仅当对象都没有任何引用时才卸载AssetBundles，这允许卸载后重新加载对象而不会造成内存重复的问题。  

如果必须使用AssetBundle.Unload（false），那么对象只能通过两种方式卸载：   

+ 在场景和代码中消除对不需要的对象的所有引用后，调用Resources.UnloadUnusedAssets。  
+ 不使用Additive方式加载新场景。 Unity将销毁当前场景中的所有对象并自动调用Resources.UnloadUnusedAssets。   
<br/>  

### Resources 和 AssetBundle 无缝切换
有时候开发初期使用Resources，后期发布时要换成 AB，就很麻烦。可以封装一个类，无缝切换二者。使用 scriptableObject 关联资源和AB的引用关系。

资源描述 ScriptableObject：  
{% highlight c# %}  
[System.Serializable]
public class BundleList : ScriptableObject {    
   public List<BundleData> bundleDatas = new List<BundleData>();

   //保存每个res路径对应的Bundle路径
   [System.Serializable]
   public class BundleData
   {
      public string resPath = string.Empty;
      public string bundlePath = string.Empty;
   }
}
{% endhighlight %} 

构建AB:  
{% highlight c# %}  
public class BuildAB  
{
   [MenuItem("Tools/BuildAssetbundle")]
   static void BuildAssetbundle()
   {
      string outPath = Path.Combine(Application.dataPath, "StreamingAssets");
      
      if (Directory.Exists(outPath)) Directory.Delete(outPath, true);
      Directory.CreateDirectory(outPath);

      List<AssetBundleBuild> builds = new List<AssetBundleBuild>
      {
         new AssetBundleBuild()
         {
            assetBundleName = "Cube.unity3d",
            assetNames = new[] {"Assets/Resources/Cube1.prefab", "Assets/Resources/Cube2.prefab"}
         }
      };

      //构建Assetbundle
      BuildPipeline.BuildAssetBundles(outPath, builds.ToArray(),
         BuildAssetBundleOptions.ChunkBasedCompression | BuildAssetBundleOptions.DeterministicAssetBundle, BuildTarget.StandaloneWindows);

      //生成资源信息文件
      BundleList bundleList = ScriptableObject.CreateInstance<BundleList>();
      foreach (var item in builds)
      {
         foreach (var res in item.assetNames)
         {
            bundleList.bundleDatas.Add(new BundleList.BundleData()
               {resPath = res, bundlePath = item.assetBundleName});
         }
      }

      AssetDatabase.CreateAsset(bundleList, "Assets/Resources/bundleList.asset");
      AssetDatabase.Refresh();
   }
}
{% endhighlight %} 

资源调用类: 
{% highlight c# %}  
public static class Assets
{
   private static readonly Dictionary<string, string> BundleDic = new Dictionary<string, string>();
   public static readonly Dictionary<string, AssetBundle> BundleCache = new Dictionary<string, AssetBundle>();

   static Assets()
   {
      //读取依赖关系
      BundleList list = Resources.Load<BundleList>("bundleList");
      foreach (var bundleData in list.bundleDatas)
      {
         Debug.Log("res: " + bundleData.resPath);
         BundleDic[bundleData.resPath] = bundleData.bundlePath;
      }
   }

   public static T LoadAsset<T>(string resName) where T : Object
   {
      string bundlePath;
      string resPath = "Assets/Resources/" + resName;
      if (typeof(T) == typeof(GameObject))
      {
         resPath = Path.ChangeExtension(resPath, "prefab");
      }
      
      //如果Bunble有这个资源从Bundle中加载.
      if (BundleDic.TryGetValue(resPath, out bundlePath))
      {
         AssetBundle assetbundle;
         if (!BundleCache.TryGetValue(bundlePath, out assetbundle))
         {
            assetbundle = AssetBundle.LoadFromFile(Path.Combine(Application.streamingAssetsPath, bundlePath));
            BundleCache[bundlePath] = assetbundle;
         }
         
         return assetbundle.LoadAsset<T>(resPath);
      }

      //如果Bundle中没有，从Resources目录中加载
      return Resources.Load<T>(resName);
   }
}
{% endhighlight %}   
<br/>  


### 资源加载策略
1. 从 Application.persistentDataPath 查找读写目录下是否有需要加载的 AB。
2. 如果1步没有加载到资源，接着在 Application.streamingAssetsPath 中查找本地是否有需要加载的AB。
3. 如果2没加载到资源，在Resources 中加载文件。
如此即可保证加载的资源永远是最新的。

#### 资源更新
将CDN上的资源下载并保存在 persistentDataPath 目录，然后按上面加载顺序加载资源。需要维护一个下载资源列表，里面记录每个资源的散列值，应用启动时，包体内资源散列值和CDN中的散列值比较，决定是否需要下载。
为避免重复下载，须保证每次相同资源创建的AB是一致的。创建方法中须设置 BuildAssetBundleOptions.DeterministicAssetBundle，可以保证打包资源的一致性。
另外，AB不能信任 MD5，Unity 提供了一个散列值来约束一致性，调用 AssetBundleManifest.GetAssetBundleHash() 来获得散列值，比较是否需要更新。

