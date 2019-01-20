---
layout: post
title: UGUI 实用技巧 2
description: "UGUI 实例总结2"
modified: 2019-01-20
tags: [Unity]
---

### UI 置灰
UI置灰也是常用的效果，直接改颜色并不能达到很好的效果，此时可以直接更换一个实现置灰效果的Shader。
而对于文字，一般都已经是灰度的了，可以跟美术商量置灰效果时具体改成要改成什么颜色。

替换材质脚本如下:  

{% highlight c# %}  
using UnityEngine;
using UnityEngine.UI;

[DisallowMultipleComponent]
public class UIGray : MonoBehaviour
{
	private static Material _grayMaterial;
	private static Material GrayMaterial
	{
		get
		{
			if (_grayMaterial == null)
				_grayMaterial = new Material(Shader.Find("UI/Gray"));
			return _grayMaterial;
		}
	}
	
	private bool isGray;
	public bool IsGray
	{
		get { return isGray; }
		set
		{
			if (isGray != value)
			{
				isGray = value;
				SetGray(isGray);
			}
		}
	}

	private Graphic[] graphics;
	void SetGray(bool gray) 
	{
		if (graphics == null || graphics.Length == 0)
		{
			graphics = transform.GetComponentsInChildren<Graphic>();
		}
		if (graphics == null) return;
		foreach (var graphic in graphics)
		{
			graphic.material = gray ? GrayMaterial : null;
		}
	}
}

#if UNITY_EDITOR
[UnityEditor.CustomEditor (typeof(UIGray))]
public class UIGrayInspector : UnityEditor.Editor 
{
	public override void OnInspectorGUI()
	{
		base.OnInspectorGUI();
		UIGray gray = target as UIGray;
		gray.IsGray = GUILayout.Toggle(gray.IsGray, " IsGray");
		if(GUI.changed)
		{
			UnityEditor.EditorUtility.SetDirty(target);
		}
	}
}
#endif
{% endhighlight %}  

置灰shader也比较简单，仍然需要更改默认UI shader，在 frag函数中利用一个常用灰度公式转换颜色
``` Gray = R * 0.299 + G * 0.587 + B * 0.114 ```

{% highlight c# %}  
fixed4 frag(v2f IN) : SV_Target
{
    ...
    float gray = dot(color.xyz, float3(0.299, 0.587, 0.114));
    color.xyz = float3(gray, gray, gray);
    return color;
}
{% endhighlight %}    
<br/>  


### 粒子裁切
有时在Mask遮罩上会有粒子效果，但是粒子并没有被限制在遮罩范围内，这时候就要通过 shader 剔除遮罩外部的粒子。
以下的脚本实现获取遮罩边界，并赋值给shader的功能。  
需要注意的是，粒子特效有很多种shader，实际使用中可能需要先实现许多不同种的粒子遮罩shader，然后根据不同情况更换对应的。

{% highlight c# %}  
using UnityEngine;
using UnityEngine.UI;

[DisallowMultipleComponent]
[RequireComponent(typeof(Image))]
[AddComponentMenu("UI/UI Particles Mask")]
public class UIParticlesMask : Mask
{
	private float minX, minY, maxX, maxY;
	private readonly Vector3[] corners = new Vector3[4];

	protected override void OnEnable()
	{
		base.OnEnable();
		Refresh();
	}

	protected override void OnRectTransformDimensionsChange()
	{
		base.OnRectTransformDimensionsChange();
		Refresh();
	}

	public void Refresh()
	{
		if (!Application.isPlaying) return;
		
		// get mask corners
		rectTransform.GetWorldCorners(corners);
		minX = corners[0].x;
		minY = corners[0].y;
		maxX = corners[2].x;
		maxY = corners[2].y;

		foreach (ParticleSystemRenderer psr in transform.GetComponentsInChildren<ParticleSystemRenderer>(true))
		{
			SetRenderer(psr);
		}
	}

	void SetRenderer(Renderer render)
	{
		if (render.sharedMaterial)
		{
			// set shader properties
			Material mat = render.material;
			mat.shader = Resources.Load<Shader>("UI Particles Mask/Alpha Blended");
			mat.SetFloat("_MinX", minX);
			mat.SetFloat("_MinY", minY);
			mat.SetFloat("_MaxX", maxX);
			mat.SetFloat("_MaxY", maxY);
		}
	}
}
{% endhighlight %}   

接着是shader内容，我们需要根据四个范围点判断像素是否显示。

{% highlight c# %}  
// 定义属性
Properties {	
    ...   
    _MinX ("Min X", Float) = -10
    _MaxX ("Max X", Float) = 10
    _MinY ("Min Y", Float) = -10
    _MaxY ("Max Y", Float) = 10
    ...
}

Pass{
    ...
    // 声明属性
    float _MinX;
    float _MaxX;
    float _MinY;
    float _MaxY;
    ...

    struct v2f {
        ...
        // 顶点着色器输出顶点位置
        float3 vpos : TEXCOORD2;
    };
    
    v2f vert (appdata_t v)
    {
        ...
        // 设置顶点位置
        o.vpos = v.vertex.xyz;        
        return o;
    }

    fixed4 frag (v2f i) : SV_Target
    {
        ...
        // 判断是否显示
        col.a *= (i.vpos.x >= _MinX );
        col.a *= (i.vpos.x <= _MaxX);
        col.a *= (i.vpos.y >= _MinY);
        col.a *= (i.vpos.y <= _MaxY);
        col.rgb *= col.a;        
        return col;
    } 
    ... 
}  
{% endhighlight %}   
<br/>  


### Scroll 嵌套
有时需要实现Scroll嵌套功能，如外部是一个横向滚动的界面，内部每个元素又是竖向滚动的元素。  
我们可以重写ScrollRect方法，在每个事件内，检测是否当前手指滑动方向与设置的滚动方向一致，如果不一致则将事件传递到父界面去。  
注意，此功能仅适用于父和子的滚动方向不一致，当一致时，有一种方法就是根据 normalizedPosition 判断子滚动界面是否到边界，然后传递给父物体，但此时事件状态仍属于子物体，直接传递给父物体会出现错误，还要进行额外的重置或者备份恢复，比较麻烦，而且一般也不会有同方向的嵌套界面需求，就不考虑此情况了。

给父和子滚动界面都使用此脚本即可： 

{% highlight c# %}  
using UnityEngine;
using UnityEngine.EventSystems;
using UnityEngine.UI;

public class CustomScrollRect : ScrollRect
{
    //父CustomScrollRect对象
    private CustomScrollRect m_Parent;

    public enum Direction
    {
        Horizontal,
        Vertical
    }

    //滑动方向
    private Direction m_Direction = Direction.Horizontal;

    //当前操作方向
    private Direction m_BeginDragDirection = Direction.Horizontal;

    protected override void Awake()
    {
        base.Awake();
        //找到父对象
        Transform parent = transform.parent;
        if (parent)
        {
            m_Parent = parent.GetComponentInParent<CustomScrollRect>();
        }

        m_Direction = this.horizontal ? Direction.Horizontal : Direction.Vertical;
    }


    public override void OnBeginDrag(PointerEventData eventData)
    {
        if (m_Parent)
        {
            m_BeginDragDirection = Mathf.Abs(eventData.delta.x) > Mathf.Abs(eventData.delta.y)
                ? Direction.Horizontal
                : Direction.Vertical;
            if (m_BeginDragDirection != m_Direction)
            {
                //当前操作方向不等于滑动方向，将事件传给父对象
                ExecuteEvents.Execute(m_Parent.gameObject, eventData, ExecuteEvents.beginDragHandler);
                return;
            }
            normalizedPosition
        }

        base.OnBeginDrag(eventData);
    }

    public override void OnDrag(PointerEventData eventData)
    {
        if (m_Parent)
        {
            if (m_BeginDragDirection != m_Direction)
            {
                //当前操作方向不等于滑动方向，将事件传给父对象
                ExecuteEvents.Execute(m_Parent.gameObject, eventData, ExecuteEvents.dragHandler);
                return;
            }
        }

        base.OnDrag(eventData);
    }

    public override void OnEndDrag(PointerEventData eventData)
    {
        if (m_Parent)
        {
            if (m_BeginDragDirection != m_Direction)
            {
                //当前操作方向不等于滑动方向，将事件传给父对象
                ExecuteEvents.Execute(m_Parent.gameObject, eventData, ExecuteEvents.endDragHandler);
                return;
            }
        }

        base.OnEndDrag(eventData);
    }

    public override void OnScroll(PointerEventData data)
    {
        if (m_Parent)
        {
            if (m_BeginDragDirection != m_Direction)
            {
                //当前操作方向不等于滑动方向，将事件传给父对象
                ExecuteEvents.Execute(m_Parent.gameObject, data, ExecuteEvents.scrollHandler);
                return;
            }
        }

        base.OnScroll(data);
    }
}
{% endhighlight %}  
<br/>  

### 不规则点击区域
有时会有需要点击区域为非矩形的不规则区域，对此，我们可以利用 PolygonCollider2D 组件来代替UI射线检测。
取消按钮与子元素的 RaycastTarget 选项, 然后增加一个子物体，挂上PolygonCollider2D组件，并编辑自己需要的形状。接着挂载检测脚本，此脚本继承Image，重写其 IsRaycastLocationValid 方法来检测当前射线是否点击中了当前的 PolygonCollider2D。

{% highlight c# %}  
using UnityEngine;
using UnityEngine.UI;

[RequireComponent(typeof(PolygonCollider2D))]
public class UIPolygon : Image
{
    private PolygonCollider2D _polygon;
    private PolygonCollider2D Polygon
    {
        get
        {
            if (_polygon == null)
                _polygon = GetComponent<PolygonCollider2D>();
            return _polygon;
        }
    }

    //设置只响应点击，不进行渲染
    protected UIPolygon()
    {
        useLegacyMeshGeneration = true;
    }

    protected override void OnPopulateMesh(VertexHelper vh)
    {
        vh.Clear();
    }

    public override bool IsRaycastLocationValid(Vector2 screenPoint, Camera eventCamera)
    {
        return Polygon.OverlapPoint(eventCamera.ScreenToWorldPoint(screenPoint));
    }

#if UNITY_EDITOR
    protected override void Reset()
    {
        //重置不规则区域
        base.Reset();
        transform.position = Vector3.zero;
        float w = (rectTransform.sizeDelta.x * 0.5f) + 0.1f;
        float h = (rectTransform.sizeDelta.y * 0.5f) + 0.1f;
        Polygon.points = new[]
        {
            new Vector2(-w, -h),
            new Vector2(w, -h),
            new Vector2(w, h),
            new Vector2(-w, h)
        };
    }
#endif
}

#if UNITY_EDITOR
[UnityEditor.CustomEditor(typeof(UIPolygon), true)]
public class UIPolygonInspector : UnityEditor.Editor
{
    public override void OnInspectorGUI()
    {
        //什么都不写用于隐藏面板的显示
    }
}
#endif
{% endhighlight %}  
<br/>  