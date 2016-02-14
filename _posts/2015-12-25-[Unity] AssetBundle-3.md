---
layout: post
title: AssetBundles：Unity 5 Asset Bundles 简介 3
description: "Unity 5 Asset Bundles 介绍：其他"
modified: 2015-12-25
tags: [Unity3D, AssetBundles]
---

###在AssetBundle中存储和加载二进制数据
第一步是使用“.bytes”扩展名保存你的二进制数据。Unity会将这个文件视为[TextAsset](http://docs.unity3d.com/ScriptReference/TextAsset.html)。一旦你下载了这个AssetBundle，并且加载了 TextAsset 对象，你就可以使用TextAsset的 .bytes 属性来获取二进制数据。

{% highlight c# %} 
string url = "http://www.mywebsite.com/mygame/assetbundles/assetbundle1.unity3d";
IEnumerator Start () {
    // Start a download of the given URL
    WWW www = WWW.LoadFromCacheOrDownload (url, 1);

    // Wait for download to complete
    yield return www;

    // Load and retrieve the AssetBundle
    AssetBundle bundle = www.assetBundle;

    // Load the TextAsset object
    TextAsset txt = bundle.Load("myBinaryAsText", typeof(TextAsset)) as TextAsset;

    // Retrieve the binary data as an array of bytes
    byte[] bytes = txt.bytes;
}
{% endhighlight %}

###内容保护
虽然使用加密技术可以确保你的Asset在传输时的安全，但一旦用户掌握了数据，他们总有办法获取到数据的内容。比如有工具可以获取发送到GPU的3D模型和贴图。因此只要用户愿意，你的资源总是会被提取到的。

当然，你还是可以使用自己的加密方法到AssetBundle文件。

一种方法是使用 TextAsset 类型来保存你的数据为字节。你可以编码你的文件然后保存为“.bytes”类型。客户端下载了AssetBundle后，从TextAsset获取到数据然后进行解密。这种方法，AssetBundle没有加密，但其存储的TextAsset数据是加过密的。

{% highlight c# %} 
string url = "http://www.mywebsite.com/mygame/assetbundles/assetbundle1.unity3d";
IEnumerator Start () {
    // Start a download of the encrypted assetbundle
    WWW www = new WWW.LoadFromCacheOrDownload (url, 1);

    // Wait for download to complete
    yield return www;

    // Load the TextAsset from the AssetBundle
    TextAsset textAsset = www.assetBundle.Load("EncryptedData", typeof(TextAsset));
 
    // Get the byte data
    byte[] encryptedData = textAsset.bytes;

    // Decrypt the AssetBundle data
    byte[] decryptedData = YourDecryptionMethod(encryptedData);

    // Use your byte array. The AssetBundle will be cached
}
{% endhighlight %}

另一种方法是从源头上，完全加密AssetBundle，然后使用WWW类下载。你可以使用任意的文件类型，只要你的服务器将其视为二进制数据。一旦下载了数据，你可以从WWW的bytes属性获取数据，然后使用自己的解密方法来获得解密后的AssetBundle文件，然后再使用 AssetBundle.CreateFromMemory 从内存创建AssetBundle。

{% highlight c# %} 
string url = "http://www.mywebsite.com/mygame/assetbundles/assetbundle1.unity3d";
IEnumerator Start () {
    // Start a download of the encrypted assetbundle
    WWW www = new WWW (url);

    // Wait for download to complete
    yield return www;

    // Get the byte data
    byte[] encryptedData = www.bytes;

    // Decrypt the AssetBundle data
    byte[] decryptedData = YourDecryptionMethod(encryptedData);

    // Create an AssetBundle from the bytes array

    AssetBundleCreateRequest acr = AssetBundle.CreateFromMemory(decryptedData);
    yield return acr;

    AssetBundle bundle = acr.assetBundle;

    // You can now use your AssetBundle. The AssetBundle is not cached.
}
{% endhighlight %}

第二种方法的好处是可以使用任意方法（除了AssetBundles.LoadFromCacheOrDownload）传输加密的数据，比如socket插件。坏处是它不会自动缓存。除了WebPlayer，你可以手动的存储文件到硬盘然后使用 AssetBundles.CreateFromFile 加载。

第三种方法结合了前两种方法，存储一个AssetBundle为TextAsset，并将其存入另外普通的AssetBundle。这个未加密的AssetBundle保存了一个会被缓存的加密的AssetBundle。原始的AssetBundle然后会被加载进内存，然后使用 AssetBundle.CreateFromMemory 获取并解密。

{% highlight c# %} 
string url = "http://www.mywebsite.com/mygame/assetbundles/assetbundle1.unity3d";
IEnumerator Start () {
    // Start a download of the encrypted assetbundle
    WWW www = new WWW.LoadFromCacheOrDownload (url, 1);

    // Wait for download to complete
    yield return www;

    // Load the TextAsset from the AssetBundle
    TextAsset textAsset = www.assetBundle.Load("EncryptedData", typeof(TextAsset));
 
    // Get the byte data
    byte[] encryptedData = textAsset.bytes;

    // Decrypt the AssetBundle data
    byte[] decryptedData = YourDecryptionMethod(encryptedData);

    // Create an AssetBundle from the bytes array
    AssetBundleCreateRequest acr = AssetBundle.CreateFromMemory(decryptedData);
    yield return acr;

    AssetBundle bundle = acr.assetBundle;

    // You can now use your AssetBundle. The wrapper AssetBundle is cached
}
{% endhighlight %}

###在AssetBundles中包含脚本
AssetBundle可以将脚本保存为TextAsset，但这样就不是可执行代码了。如果你想包含可执行代码，就需要预编译脚本并使用 Mono 反射类（反射在AOT编译平台无法使用，比如iOS）。

注意：从AssetBundle加载代码在 Windows Store Apps 和 Windows Phone 是不支持的。

{% highlight c# %} 
//c# example
string url = "http://www.mywebsite.com/mygame/assetbundles/assetbundle1.unity3d";
IEnumerator Start () {
    // Start a download of the given URL
    WWW www = WWW.LoadFromCacheOrDownload (url, 1);

    // Wait for download to complete
    yield return www;

    // Load and retrieve the AssetBundle
    AssetBundle bundle = www.assetBundle;

    // Load the TextAsset object
    TextAsset txt = bundle.Load("myBinaryAsText", typeof(TextAsset)) as TextAsset;

    // Load the assembly and get a type (class) from it
    var assembly = System.Reflection.Assembly.Load(txt.bytes);
    var type = assembly.GetType("MyClassDerivedFromMonoBehaviour");

    // Instantiate a GameObject and add a component with the loaded class
    GameObject go = new GameObject();
    go.AddComponent(type);
}
{% endhighlight %}