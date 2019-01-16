---
layout: post
title: Unity Editor 扩展编程介绍
description: "Unity Editor 扩展"
modified: 2019-01-16
tags: [Unity]
---

### MenuItem
MenuItem 用于扩展菜单选项，而且还可以拓展 Project 和 Hierarchy 窗口的菜单。

{% highlight c# %}  
// 在菜单 Window下 添加一个选项
[MenuItem("Window/My Window")]
static void MenuItemTest1() { ... }
{% endhighlight %}

{% highlight c# %}  
// 当路径为Assets，则在 Project 右键菜单 添加一个选项
[MenuItem("Assets/My Tool")]
static void MenuItemTest2() { ... }
{% endhighlight %}

{% highlight c# %}  
// 当路径为GameObject，且priority 在前面，则在 Hierarchy Create 添加一个选项
[MenuItem("GameObject/My Tool", false, 0)]
static void MenuItemTest3() { ... }
{% endhighlight %}

##### 全局自定义快捷键
在 MenuItem 路径后可以设置自定义快捷键：

    %: ctrl(windows) / command(macOS)  
    #: shift  
    &: alt  
    UP/DOWN/LEFT/RIGHT: 上下左右 四个按键
    F1...F12: F1 到 F12
    HOME, END, PGUP, PGDN

如快捷键是 Ctrl + g:  
    [MenuItem("MyMenu/Do Something with a Shortcut Key %g")]

##### 重写 Hierarchy 自带菜单功能
通过MneuItem重写同名的选项， 实现新建一个Image，并且默认不开启 raycastTarget：  
{% highlight c# %}  
[MenuItem("GameObject/UI/Image")]
static void CreatImage()
{
    Transform parent = null;
    if(Selection.activeTransform && Selection.activeTransform.GetComponentInParent<Canvas>())
    {
        parent = Selection.activeTransform;
    }
    else
    {
        Canvas canvas = Object.FindObjectOfType<Canvas>();
        if (canvas == null)
        {
            Debug.LogError("please create a canvas first!");
            return;
        }
        parent = canvas.transform;
    }

    Image image = new GameObject("image").AddComponent<Image>();
    image.raycastTarget = false;
    image.transform.SetParent(parent, false);
    Selection.activeTransform = image.transform;
}
{% endhighlight %}  
<br/>  


### 拓展 Project 和 Hierarchy 中Item的视图
主要依靠委托实现：
``EditorApplication.projectWindowItemOnGUI``：Project Window 中每个可见物体的OnGUI事件委托
``EditorApplication.hierarchyWindowItemOnGUI``：Hierarchy Window 中每个可见物体的OnGUI事件委托

``[InitializeOnLoadMethod] 属性``: 可以确保每次编译代码后首先调用。

在物体旁边加一个按钮：
![]({{ site.url }}/images/post/EditorScripts/Editor-1.jpg)

Project 窗口:
{% highlight c# %}  
[InitializeOnLoadMethod]
static void ExtensionProjectButton()
{
    EditorApplication.projectWindowItemOnGUI = (guid, rect) =>
    {
        string selectGuid = AssetDatabase.AssetPathToGUID(AssetDatabase.GetAssetPath(Selection.activeObject));
        if (Selection.activeObject && guid == selectGuid)
        {
            float width = 40;
            rect.x += rect.width - width;
            rect.width = width;
            
            if (GUI.Button(rect, "Click"))
            {
                Debug.Log("click: " + Selection.activeObject.name);
            }
        }
    };
}
{% endhighlight %}

Hierarchy 窗口:
{% highlight c# %}  
[InitializeOnLoadMethod]
static void ExtensionHierarchyButton()
{
    EditorApplication.hierarchyWindowItemOnGUI = (id, rect) =>
    {
        if (Selection.activeObject && id == Selection.activeObject.GetInstanceID())
        {
            float width = 50f;
            rect.x += rect.width - width;
            rect.width = width;
            
            // btn.png 是Asset下的一张图片
            if (GUI.Button(rect, AssetDatabase.LoadAssetAtPath<Texture>("Assets/btn.png")))
            {
                Debug.LogFormat("click : {0}", Selection.activeObject.name);
            }
        }
    };
}
{% endhighlight %}  
<br/>  


### 监听资源修改事件
继承 [UnityEditor.AssetModificationProcessor](https://docs.unity3d.com/ScriptReference/AssetModificationProcessor.html) 类， 重写监听资源的方法：  
[IsOpenForEdit](https://docs.unity3d.com/ScriptReference/AssetModificationProcessor.IsOpenForEdit.html)  
[OnWillCreateAsset](https://docs.unity3d.com/ScriptReference/AssetModificationProcessor.OnWillCreateAsset.html)  
[OnWillDeleteAsset](https://docs.unity3d.com/ScriptReference/AssetModificationProcessor.OnWillDeleteAsset.html)  
[OnWillMoveAsset](https://docs.unity3d.com/ScriptReference/AssetModificationProcessor.OnWillMoveAsset.html)  
[OnWillSaveAssets](https://docs.unity3d.com/ScriptReference/AssetModificationProcessor.OnWillSaveAssets.html)  
<br/>  

### 扩展 Inspector 窗口
Unity将许多 Editor 绘制方法都放入了dll中，为了增加自己扩展的内容，又能保持原有界面的绘制，我们可以使用反射获取dll内函数，从而实现此功能。

给Transform组件增加一个按钮：  

{% highlight c# %}  
[CustomEditor(typeof(Transform))]
public class EditorTest4 : Editor
{
    private Editor m_Editor;

    void OnEnable()
    {
        m_Editor = CreateEditor(target, Assembly.GetAssembly(typeof(Editor)).GetType("UnityEditor.TransformInspector", true));
    }

    public override void OnInspectorGUI()
    {
        if (GUILayout.Button("拓展按钮")) { }
        //调用系统绘制方法
        m_Editor.OnInspectorGUI();
    }
}
{% endhighlight %} 

![]({{ site.url }}/images/post/EditorScripts/Editor-2.jpg)  
<br/>  


### 可编辑状态
有时我们可以看到 Inspector界面是灰色不可编辑的，实现的方法有几种：  

1 设置 GUI.enabled， 如上面代码改为，则 Transform 不可编辑：
{% highlight c# %}  
GUI.enabled = false;
m_Editor.OnInspectorGUI();
GUI.enabled = true;
{% endhighlight %}   

2 设置物体或者组件的 HideFlags为 NotEditable.  
<br/>      


### Inspector 组件的 Context菜单
点击组件的设置按钮（或右键），会弹出菜单，这个菜单也可以扩展，同样使用 MenuItem：  
{% highlight c# %}  
[MenuItem("CONTEXT/Transform/New Context")]
public static void NewContext1(MenuCommand command )
{	
    //获取对象名
    Debug.Log (command.context.name);
}
{% endhighlight %}  
   
对于自己实现的脚本组件，也可以在其中直接实现相关Context功能，如下重写 Remove: 
{% highlight c# %}  
[ContextMenu("Remove Component")]
void RemoveComponent()
{
    Debug.Log("Remove Component");
    //防止代码同步错误，延迟一帧调用
    UnityEditor.EditorApplication.delayCall = () => DestroyImmediate(this);
}
{% endhighlight %}   
<br/>  


### 扩展 Scene 窗口
扩展选中的组件，继承Editor， 实现OnSceneGUI()，并在 Handles.BeginGUI() 和 Handles.EndGUI() 之间实现自定义界面：  
{% highlight c# %}  
[CustomEditor(typeof(Camera))]
public class EditorTest5 : Editor
{
    void OnSceneGUI()
    {
        Camera camera = target as Camera;
        if (camera != null)
        {
            Handles.BeginGUI();
            if (GUILayout.Button("click", GUILayout.Width(60f)))
            {
                Debug.LogFormat("click = {0}", camera.name);
            }
            Handles.EndGUI();
        }
    }
}
{% endhighlight %} 

全局扩展，实现 SceneView.onSceneGUIDelegate 委托：
{% highlight c# %}  
[InitializeOnLoadMethod]
static void InitializeOnLoadMethod()
{
    SceneView.onSceneGUIDelegate = sceneView =>
    {
        Handles.BeginGUI();
        GUI.Label(new Rect(0f, 0f, 50f, 15f), "标题");
        Handles.EndGUI();
    };
}
{% endhighlight %} 

##### [SelectionBase] 属性
给组件增加此属性，可保证在 Scene 中点击其子物体，也只能选择到此父物体。    
<br/>  

### Inspector 面板扩展
对于自定义脚本的 Inspector 面板，一般是直接用 public 可序列化变量来显示的。为了扩展，可以使用 EditorGUI 进行绘制。  
EditorGUI 和 GUI 用法基本一致，提供了大量组件，如文本、按钮、图片等。要绘制 EditorGUI, 需要调用``[CustomEditor(typeof(YourType))]``属性，并继承 Editor，重写 OnInspectorGUI 函数。  

一个完整示例：  
![]({{ site.url }}/images/post/EditorScripts/Editor-3.jpg)  

{% highlight c# %}  
using UnityEngine;
using UnityEditor;

public class EditorTest6 : MonoBehaviour
{
    public int myId;
    public string myName;
    public GameObject prefab;
    public bool toogle1;
    public bool toogle2;

    public enum MyEnum
    {
        One = 1,
        Two,
    }
    public MyEnum myEnum = MyEnum.One;
}

#if UNITY_EDITOR
[CustomEditor(typeof(EditorTest6))]
public class ScriptEditorTest6 :Editor
{
    Vector3 scrollPos = Vector3.zero;
    private bool m_EnableToogle;
    
    public override void OnInspectorGUI()
    {
        //获取脚本对象
        EditorTest6 script = target as EditorTest6;
        if(script == null) return;
        
        //绘制滚动条
        scrollPos =EditorGUILayout.BeginScrollView(scrollPos, false, true);

        script.myName = EditorGUILayout.TextField("text", script.myName);
        script.myId = EditorGUILayout.IntField("int", script.myId);
        script.prefab =
            EditorGUILayout.ObjectField("GameObject", script.prefab, typeof(GameObject), true) as GameObject;

        //绘制按钮
        EditorGUILayout.BeginHorizontal();
        if (GUILayout.Button("1"))
        {
            script.myEnum = EditorTest6.MyEnum.One;
        }
        if (GUILayout.Button("2"))
        {
            script.myEnum = EditorTest6.MyEnum.Two;
        }
        
        //绘制Enum
        script.myEnum = (EditorTest6.MyEnum) EditorGUILayout.EnumPopup("MyEnum:", script.myEnum);
        EditorGUILayout.EndHorizontal();
        
        //Toogle组件
        m_EnableToogle = EditorGUILayout.BeginToggleGroup("EnableToogle", m_EnableToogle);
        script.toogle1 = EditorGUILayout.Toggle("toogle1", script.toogle1);
        script.toogle2 = EditorGUILayout.Toggle("toogle2", script.toogle2);
        EditorGUILayout.EndToggleGroup();

        EditorGUILayout.EndScrollView();
    }
}
#endif
{% endhighlight %} 

### EditorWindow 窗口
我们也可以通过Unity Editor 实现自己的窗口，进行更丰富的交互操作。这是通过 EditorWindow 实现，其实Unity自身的窗口也是如此实现的。  
要实现窗口功能，需要继承EditorWindow类。

使用 EditorWindow.GetWindow() 可以打开窗口。

{% highlight %} 
[MenuItem("Window/My Window")]
public static void ShowWindow()
{
    //EditorWindow.GetWindowWithRect(typeof(MyWindow), new Rect(50, 100, 600, 600));
    EditorWindow.GetWindow(typeof(MyWindow), false, "My Window");
}
{% endhighlight %} 

可以在 OnGUI() 中绘制窗口。

同时可以通过 OnAwake, OnDestroy, OnFocus, OnLostFocus, OnHierarchyChange, OnInspectorUpdate, OnProjectChange, OnSelectionChange 等事件监控窗口状态。
具体查看[Editor window 官方文档](https://docs.unity3d.com/ScriptReference/EditorWindow.html)

另有本人一个[开源项目](https://github.com/byfine/UnityExcelTool)，实现了一个EXCEL 读取的窗口，可以参考。

##### 在自定义窗口中显示 Preview
有时想在自定义窗口显示预览窗口，查看模型等资源，可以通过 Editor.OnPreviewGUI 实现。

{% highlight %} 
using UnityEngine;
using UnityEditor;

public class EditorTest7 : EditorWindow
{
    private GameObject m_MyGo;
    private Editor m_MyEditor;

    [MenuItem("Window/Open My Window")]
    static void Init()
    {
        GetWindow(typeof(EditorTest7)).Show();
    }
    
    void OnGUI() {
        
        //设置一个游戏对象
        m_MyGo = (GameObject) EditorGUILayout.ObjectField(m_MyGo, typeof(GameObject), true);

        if (m_MyGo != null) {
            if (m_MyEditor == null) {
                //创建Editor实例
                m_MyEditor = Editor.CreateEditor (m_MyGo);
            }
            //预览它
            m_MyEditor.OnPreviewGUI(GUILayoutUtility.GetRect(500, 500), EditorStyles.whiteLabel);
        }
    }
}
{% endhighlight %} 