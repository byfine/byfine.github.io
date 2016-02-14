---
layout: post
title: AssetBundles：Unity 5 Asset Bundles 简介 2
description: "Unity 5 Asset Bundles 介绍：下载，加载"
modified: 2015-12-23
tags: [Unity3D, AssetBundles]
---

###下载 AssetBundles
有两种方法下载：

1. 非缓存：通过构造一个[WWW](http://docs.unity3d.com/ScriptReference/WWW.html)对象完成。AssetBundles不会缓存到本地存储设备上的Unity缓存文件夹。

2. 缓存：通过使用[WWW.LoadFromCacheOrDownload](http://docs.unity3d.com/ScriptReference/WWW.LoadFromCacheOrDownload.html)实现。AssetBundles会缓存到本地存储设备上的Unity缓存文件夹。WebPlayer共享缓存允许50MB大小的缓存。PC/Mac 和 iOS/Android 限制是4GB。

下面是一个非缓存的例子：

{% highlight c# %} 
using System;
using UnityEngine;
using System.Collections; 

class NonCachingLoadExample : MonoBehaviour {
    
    public string BundleURL;
    public string AssetName;

    IEnumerator Start() {
        // Download the file from the URL. It will not be saved in the Cache
        using (WWW www = new WWW(BundleURL)) {
            yield return www;
            
            if (www.error != null)
                throw new Exception("WWW download had an error:" + www.error);
                
            AssetBundle bundle = www.assetBundle;
            if (AssetName == "")
                Instantiate(bundle.mainAsset);
            else
                Instantiate(bundle.LoadAsset(AssetName));
                
            // Unload the AssetBundles compressed contents to conserve memory
            bundle.Unload(false);
        } 
        // memory is freed from the web stream (www.Dispose() gets called implicitly)
    }
}
{% endhighlight %}


一个使用 WWW.LoadFromCacheOrDownload 的缓存的例子：

{% highlight c# %} 
using System;
using UnityEngine;
using System.Collections;

public class CachingLoadExample : MonoBehaviour {
    public string BundleURL;
    public string AssetName;
    public int version;

    void Start() {
        StartCoroutine (DownloadAndCache());
    }

    IEnumerator DownloadAndCache (){
        // Wait for the Caching system to be ready
        while (!Caching.ready)
            yield return null;

        // Load the AssetBundle file from Cache if it exists with the same version 
        // or download and store it in the cache
        using(WWW www = WWW.LoadFromCacheOrDownload (BundleURL, version)){
            yield return www;
            
            if (www.error != null)
                throw new Exception("WWW download had an error:" + www.error);
                
            AssetBundle bundle = www.assetBundle;
            
            if (AssetName == "")
                Instantiate(bundle.mainAsset);
            else
                Instantiate(bundle.LoadAsset(AssetName));
                
            // Unload the AssetBundles compressed contents to conserve memory
            bundle.Unload(false);

        } // memory is freed from the web stream (www.Dispose() gets called implicitly)
    }
}
{% endhighlight %}

当你获取 .assetBundle 资源，下载的数据被提取并且 AssetBundle 对象会被创建。这时，你就可以加载bundle中包含的对象。LoadFromCacheOrDownload 第二个参数指定了要下载的AssetBundle的版本。如果缓存中没有或者版本比需要的低，就会下载AssetBundle。否则就直接从缓存中获取AssetBundle。

####Editor环境下加载
Editor下加载AssetBundles比较麻烦，因为有资源更新时，还要重新打包AssetBundles。这时候通过 Resources.LoadAssetAtPath 直接获取Asset比较方便：
 
{% highlight c# %} 
// C# Example
// Loading an Asset from disk instead of loading from an AssetBundle
// when running in the Editor
using System.Collections;
using UnityEngine;

class LoadAssetFromAssetBundle : MonoBehaviour
{
    public Object Obj;

    public IEnumerator DownloadAssetBundle<T>(string asset, string url, int version) where T : Object {
        Obj = null;

#if UNITY_EDITOR
        Obj = Resources.LoadAssetAtPath("Assets/" + asset, typeof(T));
        yield return null;

#else
        // Wait for the Caching system to be ready
        while (!Caching.ready)
            yield return null;

        // Start the download
        using(WWW www = WWW.LoadFromCacheOrDownload (url, version)){
            yield return www;
            
            if (www.error != null)
                throw new Exception("WWW download:" + www.error);
                
            AssetBundle assetBundle = www.assetBundle;
            
            Obj = assetBundle.LoadAsset(asset, typeof(T));
            
            // Unload the AssetBundles compressed contents to conserve memory
            bundle.Unload(false);

        } // memory is freed from the web stream (www.Dispose() gets called implicitly)

#endif
    }
}
{% endhighlight %}


###加载和卸载AssetBundle中的对象

当从下载的数据创建了AssetBundle对象后，有三种方法加载AssetBundle中的资源：

- [AssetBundle.LoadAsset](http://docs.unity3d.com/ScriptReference/AssetBundle.LoadAsset.html)，会使用名字作为参数加载物体，还有一个可选的参数指定加载类型。

- [AssetBundle.LoadAssetAsync](http://docs.unity3d.com/ScriptReference/AssetBundle.LoadAssetAsync.html)，效果和上面一样，只是它是异步的，加载时不会阻塞主线程。这对加载大资源或很多资源很有效。

- [AssetBundle.LoadAllAssets](http://docs.unity3d.com/ScriptReference/AssetBundle.LoadAllAssets.html)，会加载AssetBundle中的所有物体，可以指定对象类型。


要卸载资源，需要使用 AssetBundle.Unload。这个方法有一个布尔参数，以指示是卸载所有数据（包括以加载的数据），还是只卸载下载的压缩文件。如果你正在使用来自AssetBundle的一些资源，而你又想释放一些内存，可以传递参数false来卸载压缩的资源文件。而如果你不再使用AssetBundle的任何资源，可以传递true来彻底卸载已经加载的资源。

####异步加载AssetBundle的资源

{% highlight c# %} 
using UnityEngine;

// Note: This example does not check for errors. Please look at the example in the DownloadingAssetBundles section for more information
IEnumerator Start () {
    // Start a download of the given URL
    WWW www = WWW.LoadFromCacheOrDownload (url, 1);

    // Wait for download to complete
    yield return www;

    // Load and retrieve the AssetBundle
    AssetBundle bundle = www.assetBundle;

    // Load the object asynchronously
    AssetBundleRequest request = bundle.LoadAssetAsync ("myObject", typeof(GameObject));

    // Wait for completion
    yield return request;

    // Get the reference to the loaded object
    GameObject obj = request.asset as GameObject;

    // Unload the AssetBundles compressed contents to conserve memory
    bundle.Unload(false);

    // Frees the memory from the web stream
    www.Dispose();
}
{% endhighlight %}

###记录加载的AssetBundle
对于一个AssetBundle，Unity只允许同一时间加载一个实例。这意味着如果一个AssetBundle已经加载过了，且没有卸载，你就不能再用WWW获取它了。即当你试图使用如下方式获取一个已经加载过的AssetBundle：

     AssetBundle bundle = www.assetBundle;
     
这会报错：Cannot load cached AssetBundle. A file of the same name is already loaded from another AssetBundle。
而且assetBundle会返回null。如果第一个资源还在加载中时，你是不能在第二个下载时获得AssetBundle的，因此你要做的要么就是在不需要时卸载AssetBundle，要么对它保持引用以避免当它已经存在于内存中时还下载它。

要注意再Unity5之前，在要卸载bundles之前，所有bundles的加载都要完成。

一般还是推荐在不需要时卸载AssetBundle，如果你一定要记录下载的AssetBundle，你可以使用一个包装类，如下所示：

{% highlight c# %} 
using UnityEngine;
using System;
using System.Collections;
using System.Collections.Generic;

static public class AssetBundleManager {
    
   // A dictionary to hold the AssetBundle references
   static private Dictionary<string, AssetBundleRef> dictAssetBundleRefs;
   
   static AssetBundleManager (){
       dictAssetBundleRefs = new Dictionary<string, AssetBundleRef>();
   }
   
   // Class with the AssetBundle reference, url and version
   private class AssetBundleRef {
       public AssetBundle assetBundle = null;
       public int version;
       public string url;
       public AssetBundleRef(string strUrlIn, int intVersionIn) {
           url = strUrlIn;
           version = intVersionIn;
       }
   };
   
   // Get an AssetBundle
   public static AssetBundle getAssetBundle (string url, int version){
       string keyName = url + version.ToString();
       AssetBundleRef abRef;
       
       if (dictAssetBundleRefs.TryGetValue(keyName, out abRef))
           return abRef.assetBundle;
       else
           return null;
   }
   
   // Download an AssetBundle
   public static IEnumerator downloadAssetBundle (string url, int version){
       string keyName = url + version.ToString();
       if (dictAssetBundleRefs.ContainsKey(keyName))
           yield return null;
       else {
           using(WWW www = WWW.LoadFromCacheOrDownload (url, version)){
               yield return www;
               if (www.error != null)
                   throw new Exception("WWW download:" + www.error);
               AssetBundleRef abRef = new AssetBundleRef (url, version);
               abRef.assetBundle = www.assetBundle;
               dictAssetBundleRefs.Add (keyName, abRef);
           }
       }
   }
   
   // Unload an AssetBundle
   public static void Unload (string url, int version, bool allObjects){
       string keyName = url + version.ToString();
       AssetBundleRef abRef;
       if (dictAssetBundleRefs.TryGetValue(keyName, out abRef)){
           abRef.assetBundle.Unload (allObjects);
           abRef.assetBundle = null;
           dictAssetBundleRefs.Remove(keyName);
       }
   }
}
{% endhighlight %}

使用上面的类：

{% highlight c# %} 
using UnityEditor;

class ManagedAssetBundleExample : MonoBehaviour {
   public string url;
   public int version;
   AssetBundle bundle;
   
   void OnGUI (){
       if (GUILayout.Label ("Download bundle"){
           bundle = AssetBundleManager.getAssetBundle (url, version);
           if(!bundle)
               StartCoroutine (DownloadAB());
       }
   }
   
   IEnumerator DownloadAB (){
       yield return StartCoroutine(AssetBundleManager.downloadAssetBundle (url, version));
       bundle = AssetBundleManager.getAssetBundle (url, version);
   }
   
   void OnDisable (){
       AssetBundleManager.Unload (url, version);
   }
}
{% endhighlight %}


要注意AssetBundleManager是静态类，你记录的任何AssetBundles在加载新场景后都不会被销毁。这个类只是个参考，还是推荐一但不使用就马上卸载AssetBundles。