---
layout: post
title: Unity3D MoveTowards、Lerp、Slerp
description: "Unity3D中 MoveTowards、Lerp、Slerp 讲解"
modified: 2015-09-17
tags: [Unity3D]
---

MoveTowards、Lerp、Slerp这三个函数相信大家经常遇到，这些都是在做一些过渡操作时需要用到的，那么它们间的具体差别是什么呢？其实要搞清楚它们的区别，只要仔细看官方说明，明白它们的具体用途是什么。

## MoveTowards

Mathf、Vector2、Vector3等许多类都有这个方法，意思都差不多。以Vector3为例，函数原型为：

    public static Vector3 MoveTowards(Vector3 current, Vector3 target, float maxDistanceDelta);

作用是将当前值current移向目标target。（对Vector3是沿两点间直线）
maxDistanceDelta就是每次移动的最大长度。
返回值是当current值加上maxDistanceDelta的值，如果这个值超过了target，返回的就是target的值。

例子：

    //表示以每秒moveMax的速度从Current移动到Target。
    //因为Current和Target距离是4，所以当moveMax 等于0.5f，用时8秒，moveMax等于2时，用时2秒。
    Vector3 Current  = new Vector3(0, 0, 0);
    Vector3 Target = new Vector3(0, 0, 4);
    Current = Vector3.MoveTowards(Current, Target, moveMax * Time.deltaTime);


## Lerp

Lerp表示线性插值，很多地方都用到了它。这里仍以Vector3为例，函数原型是：

    public static Vector3 Lerp(Vector3 a, Vector3 b, float t);

它的作用就是按照t计算a和b之间的插值。t的取值范围是[0, 1]。
简单理解就是当t=0， 返回值是a，当t=1，返回值是b。同理，t=0.1时是a到b的10%。t就代表了a到b的百分比。

![线性插值图示]({{ site.url }}/images/post/2015-09-17/Unity-MoveTowardsLerp-1.png)

例1：

    //在time时间内移动物体
    private IEnumerator MoveObject(Vector3 startPos, Vector3 endPos, float time)
    {        
            var dur = 0.0f;
            while (dur <= time)
            {
                dur += Time.deltaTime;
                transform.position = Vector3.Lerp(startPos, endPos, dur / time);
                yield return null;
            }
    }

例2：

    //以指定速度speed移动物体
    private IEnumerator MoveObject_Speed(Vector3 startPos, Vector3 endPos, float speed)
    {
            float startTime = Time.time;
            float length = Vector3.Distance(startPos, endPos);
            float frac = 0;

            while (frac < 1.0f)
            {
                float dist = (Time.time - startTime) * speed;
                frac = dist / length;
                transform.position = Vector3.Lerp(startPos, endPos, frac);
                yield return null;
            }
    }


## Slerp 

球形插值在Vector3、Quaternion等类都有使用，一般多在Quaternion的旋转操作时使用。

#### 对于Vector3：

    public static Vector3 Slerp(Vector3 a, Vector3 b, float t);

这里球形插值与线性插值不同的地方在于，它将Vectors视为**方向**而不再是点。返回的向量方向，它的角度是根据a和b的角度插值，而它的长度是根据a和b的长度插值。
可以用其模拟太阳的升降变化等操作。

#### 对于Quaternion：

    public static Quaternion Slerp(Quaternion a, Quaternion b, float t);

计算a与b的球形插值。
Quaternion的Lerp与Slerp结果都是一样的，Lerp的效率会比Slerp高些，但是当旋转值a和b离得比较远时，Lerp的效果会非常差。
这是一段Lerp和Slerp的视频演示：https://www.youtube.com/watch?v=uNHIPVOnt-Y

或者看下图：

![Lerp与Slerp]({{ site.url }}/images/post/2015-09-17/Unity-MoveTowardsLerp-2.png)

红点是起点，绿点是终点。蓝色线是Lerp的轨迹，白线是Slerp的轨迹。


## 注意：
对于插值（Slerp和Lerp）运算中的t，要明白它代表的是百分比，直接对齐赋值时间如Time.deltaTime是没意义的，因为如果t为1的话，永远都不会到达目标值。