---
layout: post
title: UGUI 实用技巧 1
description: "UGUI 实例总结1"
modified: 2019-01-19
tags: [Unity]
---

### ScrollRect 制作摇杆
使用 ScrollRect 组件可以很容易的制作摇杆。
新建脚本继承ScrollRect，重写OnDrag，在其中检测摇杆距离。
注意要同时开启 vertical 和 horizontal 两个方向。
完整脚本如下:  

{% highlight c# %}  
using UnityEngine;
using UnityEngine.EventSystems;
using UnityEngine.UI;

public class Joystick : ScrollRect
{
    public Vector2 Pos;
    private Scrollbar bar;
    protected float mRadius = 0f;
    private bool isDragging;    
    
    protected override void Start()
    {
        base.Start();
        // 摇杆块的半径 
        mRadius = ((RectTransform) transform).sizeDelta.x * 0.5f;
    }

    public override void OnDrag(PointerEventData eventData)
    {
        base.OnDrag(eventData);
        isDragging = true;

        // 限制在摇杆块范围内
        var contentPostion = this.content.anchoredPosition;
        if (contentPostion.magnitude > mRadius)
        {
            contentPostion = contentPostion.normalized * mRadius;
            SetContentAnchoredPosition(contentPostion);
        }
        // 更新位置
        Pos = contentPostion.normalized;
    }

    public override void OnEndDrag(PointerEventData eventData) 
    { 
        base.OnEndDrag(eventData);
        isDragging = false;
        checkTime = Time.time;
    }

    private float checkTime;
    private void Update()
    {
        // 松手后，更新回到中心过程的位置
        if (isDragging) return;
        
        if (Time.time - checkTime < 1)
        {
            Pos = new Vector2(content.anchoredPosition.x / mRadius, content.anchoredPosition.y / mRadius);
        }
        else
        {
            Pos = Vector2.zero;
        }
    }
}
{% endhighlight %}  
<br/>  


### UI点击渗透
有时好几个UI组件叠在一起，希望点击最上层，下面的也能响应到。
继承事件监听接口，在实现事件中，都用EventSystem.current.RaycastAll()方法找的所有可以响应的对象，最后用 ExecuteEvents.Execute() 响应事件。

{% highlight c# %}  
using UnityEngine;
using UnityEngine.EventSystems;
using System.Collections.Generic;

public class MultiLayerEvent : MonoBehaviour, IPointerClickHandler, IPointerDownHandler, IPointerUpHandler
{
    public void OnPointerDown(PointerEventData eventData)
    {
        PassEvent(eventData, ExecuteEvents.pointerDownHandler);
    }

    public void OnPointerUp(PointerEventData eventData)
    {
        PassEvent(eventData, ExecuteEvents.pointerUpHandler);
    }
    
    public void OnPointerClick(PointerEventData eventData)
    {
        PassEvent(eventData, ExecuteEvents.submitHandler);
        PassEvent(eventData, ExecuteEvents.pointerClickHandler);
    }

    //把事件透下去
    public void PassEvent<T>(PointerEventData data, ExecuteEvents.EventFunction<T> function)
        where T : IEventSystemHandler
    {
        List<RaycastResult> results = new List<RaycastResult>();
        EventSystem.current.RaycastAll(data, results);
        GameObject current = data.pointerCurrentRaycast.gameObject;
        for (int i = 0; i < results.Count; i++)
        {
            if (current != results[i].gameObject)
            {
                ExecuteEvents.Execute(results[i].gameObject, data, function);
                // 如果你只想响应穿透的第一个，直接break就行
                //break;
            }
        }
    }
}

{% endhighlight %}  
<br/>  


### 用于新手引导的遮罩
新手引导经常需要一种效果，就是目标物体透明，周围遮住的遮罩：  
![]({{ site.url }}/images/post/UGUI-Utilities/UGUI-UT-1.png)  

一个简单的做法就是，在前面放一个覆盖整个屏幕大小的图片遮罩，利用上面的事件渗透方法检测点击，然后修改默认的 UI shader，根据像素点与目标的距离判断是否显示。

首先是脚本文件，用于更新shader信息。

{% highlight c# %}  
using UnityEngine;
using UnityEngine.UI;

public class Script_05_11 : MonoBehaviour
{
	public RectTransform CanvasRt;
	public Camera UICamera;

	public bool ShrinkAnim = true;
	public bool FollowTarget;
	
	private float radius;
	private float range;
	private Vector3 center;
	private readonly Vector3[] corners = new Vector3[4]; 

	private Material material;
	private RectTransform target;
	
	void Awake ()
	{
		material = GetComponent<Image>().material;
	}

	public void ResetTarget(RectTransform rt)
	{
		if (rt == null) return;
		target = rt;		
		UpdateCenter();
		UpdateRange();
	}

	// 更新位置
	void UpdateCenter()
	{
		if (target == null) return;
		
		target.GetWorldCorners(corners);
		radius = Vector2.Distance(corners[0], corners[2]) / 2f;

		float x = corners[0].x + (corners[3].x - corners[0].x) / 2f;
		float y = corners[0].y + (corners[1].y - corners[0].y) / 2f;
		center = new Vector3(x, y, 0);
		Vector2 centerPos;
		RectTransformUtility.ScreenPointToLocalPointInRectangle(CanvasRt, center, UICamera, out centerPos);

		material.SetVector("_Center", new Vector4(centerPos.x, centerPos.y, 0, 0));
	}

	// 缩小动画
	void UpdateRange()
	{
		if (!ShrinkAnim) return;
		CanvasRt.GetWorldCorners(corners);
		foreach (var c in corners)
		{
			range = Mathf.Max(Vector3.Distance(c, center), range);
		}

		material.SetFloat("_Silder", range);
		needShrink = true;
	}

	private float velocity;
	private bool needShrink;
	void Update()
	{
		if (ShrinkAnim && needShrink)
		{
			float value = Mathf.SmoothDamp(range, radius, ref velocity, 0.3f);
			if (!Mathf.Approximately(value, range))
			{
				range = value;
				material.SetFloat("_Silder", range);
			}
			else
			{
				needShrink = false;
			}
		}
		else if(FollowTarget)
		{
			UpdateCenter();
		}
	}
}
{% endhighlight %}  

调用 ResetTarget() 更新目标，目前只实现了实时更新位置，如果需要，也可以实时更新大小。  
接着是 shader 文件，我们可以从官网上下载内置的shader，UI默认的是 UI-Default.shader 文件，只需要进行简单的修改即可。

首先在 Properties 增加参数
{% highlight c# %} 
Properties
{
    ...
    _Center("Center", vector) = (0, 0, 0, 0)
    _Silder ("_Silder", Range (0,1000)) = 1000
}
{% endhighlight %} 

然后不要忘记在 Pass 内声明，最后在 frag 函数里增加两句，根据distance 函数判断距离，从而决定像素点是否透明。 
{% highlight c# %}  
Pass
{
    Name "Default"
    ...

    float _Silder;
    float2 _Center;

    ...

    fixed4 frag(v2f IN) : SV_Target
    {
        ... 
        
        color.a *= (distance(IN.worldPosition.xy,_Center.xy) > _Silder);
        color.rgb *= color.a;            
        return color;        
    }
}  
{% endhighlight %} 

<br/>  