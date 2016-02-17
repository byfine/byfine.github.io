---
layout: post
title: AssetBundles：Unity 5 Asset Bundles 简介 1
description: "Unity 5 Asset Bundles 介绍：构建、压缩"
modified: 2015-12-21
tags: [Unity3D, AssetBundles]
---

### 创建及导出 Asset Bundle

#### 设置名称
Unity5的AssetBundle系统和4有些差别，现在创建AssetBundle，
首先，你需要选中资源，然后在inspector面板最下面设置bundle名称，名称可以使用"/"来分层：

![]({{ site.url }}/images/post/AssetBundle/AB1.png)

之后对于要导出的资源，只需要选择你创建的名称即可。

当然，也可以通过代码设置名称，从而实现自动化工具，只需设置这个属性即可：
    
    AssetImporter.assetBundleName



#### 导出AB
导出时，需要在代码操作：

    using UnityEditor;

    public class CreateAssetBundles
    {
        [MenuItem ("Assets/Build AssetBundles")]
        static void BuildAllAssetBundles ()
        {
            BuildPipeline.BuildAssetBundles ("Assets/AssetBundles");
        }
    }

这会在Asset菜单创建一个按钮，点击后开始创建AssetBundle。

BuildPipeline.BuildAssetBundles函数的原型为：

    public static AssetBundleManifest BuildAssetBundles(string outputPath, BuildAssetBundleOptions assetBundleOptions = BuildAssetBundleOptions.None, BuildTarget targetPlatform = BuildTarget.WebPlayer);

第一个参数为输出路径，第二个为构建选项，第三个为构建目标平台。

注意，创建之前需要先创建好输出文件夹。

你会发现，创建好之后，每个AssetBundle会对应生成一个.manifest文件，其中包含了校验、依赖文件等信息。     
另外还会生成两个文件：AssetBundle 和 AssetBundle.manifest 文件，这会在每个生成Assetbundle的文件夹都自动创建。因此，如果你一直输出到一个文件夹，那就只会创建这两个文件。AssetBundle.manifest 文件会包含一些AssetBundles的信息和依赖关系。


#### AssetBundle变型
这是Unity5中的新特性，实现类似虚拟资源的效果。

比如你可以实现一个变型 “MyAssets.hd”和“MyAssets.sd”。构建时两个变型AssetBunlde会有相同的内部ID，而在运行时只要通过不同的扩展变型名，就可以任意切换两个资源。

设置方法是，先设置名称，然后在名称旁边的下拉菜单设置变型名。

![]({{ site.url }}/images/post/AssetBundle/AB2.png)


### Asset Bundle 压缩
Unity支持三种压缩方式：LZMA, LZ4, 和不压缩。

LZMA压缩后体积最小，但解压缩需要一定时间因此加载时较慢。LZ4压缩包较大，但解压缩时间消耗很少，另外只在5.3以后版本可以使用。不压缩是占空间最大的，但当然相对的是读取速度最快的。

#### 压缩包的缓存
[WWW.LoadFromCacheOrDownload](http://docs.unity3d.com/ScriptReference/WWW.LoadFromCacheOrDownload.html)函数会下载并缓存AssetBundle到硬盘，以便之后使用更加快速。从5.3开始，缓存的数据也可以使用LZ4算法被压缩，这会比不压缩节省40%–60%的空间。
随着数据从网络下载，Unity会解压它们并使用LZ4再压缩，压缩过程发生在下载流期间，这意味着一旦获取到足够的数据缓存就会开始压缩，并持续递增直到下载完成。之后，会通过即时解压来从缓存的bundle获取所需的数据。

缓存的压缩是默认开启的，并由[Caching.compressionEnabled](http://docs.unity3d.com/ScriptReference/Caching-compressionEnabled.html)属性控制。     
#### AssetBundle加载API
下表列出了使用不同压缩类型和加载方法时内存和性能开销的对比。

![]({{ site.url }}/images/post/AssetBundle/AB3.png)

*注意：当使用WWW下载一个bundle，WebRequest还会有一个8*64KB的缓存区来存储来自socket的数据。

参考上表，可以使用下面的规则来使用加载API：

1. 部署bundle作为流媒体资源（StreamingAssets）：构建时使用 BuildAssetBundleOptions.ChunkBasedCompression，并且加载时使用 AssetBundle.LoadFromFileAsync。这会使你的数据有一定压缩，并且在加载时获得最大可能的加载速度，以及和读取缓存一样的内存开销。

2. Asset bundles 作为下载内容：使用默认设置（LZMA压缩）构建，并且使用 LoadFromCacheOrDownload/WebRequest 下载并缓存。这样你会有最大压缩率，以及接下来加载时 AssetBundle.LoadFromFile 的加载效率。

3. 加密过的bundles：选择 BuildAssetBundleOptions.ChunkBasedCompression 并且使用 LoadFromMemoryAsync 来加载。（这几乎是 LoadFromMemory[Async] 唯一的使用情景）。

4. 自定义压缩：使用 BuildAssetBundleOptions.UncompressedAssetBundle 构建，并在用你自己的解压缩算法解压后，使用 AssetBundle.LoadFromFileAsync 加载。

一般你应该使用异步方法 - 它们不会阻塞主线程并使加载操作更有效率。并且绝对要避免同时使用同步和异步加载方法 - 这会引起主线程停顿。


### Asset Bundle 的内部结构
Asset Bundle实质是一些物体组合在一起形成的序列化文件的集合。在部署时根据它是数据bundle还是场景bundle，结构上有一些不同。

#### 普通Asset Bundle结构

![]({{ site.url }}/images/post/AssetBundle/AB4.png)

#### Streamed scene Asset Bundle结构

![]({{ site.url }}/images/post/AssetBundle/AB5.png)

#### Asset Bundle 的压缩

![]({{ site.url }}/images/post/AssetBundle/AB6.png)

如上所示，可能会有块压缩或流压缩两种方式。块压缩（LZ4）意味着原始数据被分成同样大小的块而这些块被分别压缩。如果你需要实时解压，应该使用这种方法 - 随机读取的消耗很小。而流压缩（LZMA）使用同样的字典处理整个数据块，它会提供最高的压缩率但是只支持顺序的读取。