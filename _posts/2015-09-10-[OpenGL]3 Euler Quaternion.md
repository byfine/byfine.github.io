---
layout: post
title: 欧拉角、四元数 简介
description: "欧拉角、四元数 相关数学知识"
modified: 2015-09-10
tags: [OpenGL, Math]
---

##方位、方向、角位移：

- 方位：描述的是物体的朝向。要确定一个方位（orientation），却至少需要需要三个参数。
- 方向：“方向”和“方位”并不完全一样。向量有“方向”但没有“方位”，因为让向量自转，但向量却不会有任何变化。只要用两个数字（例如：极坐标），就能用参数表示一个方向（direction）。
- 角位移：我们描述物体位置时并不是绝对坐标，而是描述相对于给定参考点的位移。同样，描述物体方位时，也不能使用绝对量。方位是通过与相对已知方位（通常称为"单位"方位或"源"方位）的旋转来描述的。旋转的量称作**角位移**。在数学上描述方位就等价于描述角位移。


##欧拉角：

欧拉角是描述方位的一种方法。

####什么是欧拉角：

欧拉角将方位（角位移）分解为绕三个互相垂直轴的旋转。任意三个轴和任意顺序都可以，但最有意义的是使用笛卡尔坐标系并按一定顺序所组成的旋转序列。		
最常用的约定是所谓的"heading-pitch-bank"约定。它的基本思想是让物体开始于"标准"方位--就是物体坐标轴和惯性坐标轴对齐。在标准方位上，让物体作heading、pitch、bank旋转，最后物体到达我们想要描述的方位。
    
首先，heading为绕y轴的旋转量，向右旋转为正：	

heading:

![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-1.png)		
    
经过heading旋转，pitch为绕x轴的旋转量，注意是物体坐标系的x轴，不是原惯性坐标系的x轴：	

pitch:

![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-2.png)	
    
经过了heading和pitch，bank为绕z轴的旋转量，注意是物体坐标系的z轴		

bank:

![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-3.png)		
    
**roll-pitch-yaw**：

heading-pitch-bank也叫做roll-pitch-yaw，roll等价于bank，yaw基本上等价于heading。注意，它的顺序和heading-pitch-bank的顺序相反，这只是语义上的。
		
![roll-pitch-yaw]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-4.png)	

注：1、旋转可以以不同的顺序进行。2、决定每个旋转的正方向时不一定必须遵守左手或右手法则。
		
####欧拉角的优点：
欧拉角仅使用三个数来表达方位，并且这三个数都是角度。这使欧拉角具有一些独特的优点：
1、欧拉角对我们来说很容易使用。2、最简洁的表达方式。3、任意三个数都是合法的。
		
####欧拉角的缺点：
1. 给定方位的表达方式不唯一。
    - 在将一个角度加上360度的倍数时，并不会改变方位。
    - 还有一种情况，由三个角度不互相独立而导致的。例如，pitch135度等价于heading180度，pitch45度，然后bank180度。为了保证任意方位都只有独一无二的表示，必须限制角度的范围。一种常用的技术是将heading和bank限制在+180度到-180度之间，pitch限制在+90度到-90度之间。
    - 万向锁：
        欧拉角最著名的别名问题是这样的：先heading45度再pitch90度，这与先pitch90度再bank45度是等价的。事实上，一旦选择+(-)90度为pitch角，就被限制在只能绕竖直轴旋转。这种现象，角度为+(-)90度的第二次旋转使得第一次和第三次旋转的旋转轴相同，称作**万向锁**。为了消除限制欧拉角的这种别名现象，规定万向锁情况下由heading完成绕竖直轴的全部旋转。换句话说，在限制欧拉角中，如果pitch为+(-)90度，则bank为0。		
        		
2. 两个角度间求插值非常困难。
    - 如果没有使用限制欧拉角，方位A的heading为720度，方位B的heading为45度，heading值只相差45度，但简单的插值会在错误的方向上绕将近两周。			
    - 设A的heading为-170度，B的heading为170度。这两个值只相差20度，但插值操作又一次发生了错误，旋转是沿 "长弧"绕了340度而不是更短的20度。		

    解决这类问题的方法是将插值的"差"角度折到-180度到180度之间，以找到最短弧。

    		
##四元数：
四元数通过使用四个数来表示方位。可以避免万向节死锁这样的问题。

####四元数记法：
一个四元数包含一个标量和一个3D向量，经常记标量分量为w，记向量分量为单一的 **v** 或分开的x、y、z。
		
####四元数与复数：
复数对(a, b)定义了数a+bi，i是虚数，满足i^2 = -1，a称作实部，b称作虚部。任意实数k都能表示为复数k + 0i。
复数集存在于一个2D平面上，可以认为这个平面有两个轴：实轴和虚轴。这样，就能将复数（x, y）解释为2D向量。用这种方法解释复数，可以表达平面中的旋转：	

![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-5.png)

旋转后的复数p'能用复数乘法计算出来：

    p = x + yi
    q = cosθ + i sinθ
    p' = pq = (x + yi)(cosθ + i sinθ) = (xcosθ - ysinθ) + (xsinθ + ycosθ)i

这和用2x2旋转矩阵达到的效果是一样的。

四元数扩展了复数系统，它使用三个虚部i, j, k。它们的关系如下：

![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-6.png)

一个四元数[w, (x, y, z)]定义了复数 w+xi+yj+zk，很多标准复数的性质都能应用到四元数上。更重要的是，和复数能用来旋转2D中的向量类似，四元数也能用来旋转3D中的向量。
	
####四元数和轴-角对：
欧拉证明了一个旋转序列等价于单个旋转。因此，3D中的任意角位移都能表示为绕单一轴的单一旋转。这种描述方位的形式称作轴-角描述法。

设n为旋转轴，将n定义为单位长度。根据左手或右手法则， n的方向定义了哪边将被认为是旋转"正"方向。设θ为绕轴旋转的量，因此，轴-角对（n , θ）定义了一个角位移：绕 n 指定的轴旋转θ角。

四元数能被解释为角位移的轴-角对方式。然而， n和θ不是直接存储在四元数的四个数中。公式列出了四元数中的数和 n，θ的关系，两种四元数加法都被使用了。

![n，θ 的关系]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-7.png)	

####负四元数：
四元数能求负，做法很直接，将每个分量都变负：
![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-8.png)		

q 和-q 代表的实际角位移是相同的。如果我们将θ加上360度的倍数，不会改变q代表的角位移，但它使q的四个分量都变负了。因此，3D中的任意角位移都有两种不同的四元数表示方法，它们互相为负。
	
####单位四元数：
几何上，存在两个"单位"四元数，它们代表没有角位移，[1, **0**]和[-1, **0**]（注意粗体**0**，它们代表零向量）。
当θ是360度的偶数倍时，cos(θ/2)=1；θ是360度的奇数倍时，cos( θ /2)=-1。在两种情况下，都有sin(θ/2)=0，所以 n 的值无关紧要。它的意义在于：
当旋转角θ是360度的整数倍时，方位并没有改变，并且旋转轴也是无关紧要的。

数学上，实际只有一个单位四元数：[1, **0**]。用任意四元数 q 乘以单位四元数[1, **0**]，结果仍是 q 。任意四元数 q 乘以另一个"几何单位"[1, **0**]时得到-q 。几何上，因为 q 和-q 代表的角位移相同，可认为结果是相同的。但在数学上， q 和-q 不相等，所以[1, **0**]并不是"真正"的单位四元数。
	
####四元数的模：
和复数一样，四元数也有模。	
![四元数的模]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-9.png)	
	
让我们看看它的几何意义，代入 θ 和 n ，可得到：	
![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-10.png)	

n 为单位向量，所以：	
![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-11.png)	

由三角公式sin2x + cos2x = 1，可得：	
![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-12.png)	

如果用四元数来表示方位，我们仅使用符合这个规则的 **单位四元数**。
	
####四元数共轭和逆：
四元数的共轭记作 q*，可通过让四元数的向量部分变负来获得：	
![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-13.png)	

四元数的逆记作 q ^-1，定义为四元数的共轭除以它的模：	
![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-14.png)	

一个四元数 q 乘以它的逆 q ^-1，即可得到单位四元数[1, **0**]。

因为我们只使用单位四元数，所以四元数的逆和共轭是相等的。
q 和 q\* 代表**相反的角位移**。很容易验证这种说法，使 v 变负，也就是使旋转轴反向，它颠倒了我们所认为的旋转正方向。因此， q 绕轴旋转θ角，而 q\* 沿相反的方向旋转相同的角度。

####四元数乘法：
四元数能根据复数乘法解释来相乘，使用分配律即可：	
![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-15.png)		
![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-16.png)	

四元数乘法满足结合律，但不满足交换律

四元数乘积的模等于模的乘积：	
![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-17.png)	

它保证了两个单位四元数相乘的结果还是单位四元数。

四元数乘积的逆等于各个四元数的逆以相反的顺序相乘：	
![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-18.png)
	  
现在来看四元数一个非常有用的性质。"扩展"一个标准3D点(x, y, z)到四元数空间，通过定义四元数p=[0, (x, y, z)]即可。设q为旋转四元数形式[cos(θ/2), nsin(θ/2)]，n为旋转轴，单位向量，θ为旋转角。则执行下面的乘法可以使3D点p绕n旋转：	
![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-19.png)	

考虑多次旋转的情况，将点p用一个四元数a旋转然后再用另一个四元数b旋转：  
![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-20.png)	

可以看到，先进行a旋转再进行b旋转，等价于ba代表的单一旋转。
因此四元数乘法能用来连接多次旋转，这和矩阵乘法的效果一样。
	
####四元数的"差"：
"差"被定义为一个方位到另一个方位的角位移。即给定方位a和b，能够计算从a旋转到b的角位移d：
ad=b
两边同时左乘a^-1:	
![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-21.png)	

现在我们有了求得代表一个方位到另一个方位角位移的四元数的方法。
两个四元数之间的角度"差"更类似于"除"。

####四元数点乘：
四元数也有点乘运算，它的记法、定义和向量点乘非常类似：	
![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-22.png)	

和向量点乘一样，其结果是标量。对于单位四元数a和b，有-1 ≤ a . b ≤ 1。
四元数点乘的几何解释类似于向量点乘的几何解释，四元数点乘 a · b 的绝对值越大，a和b代表的角位移越"相似"。
	
####四元数的对数、指数和标量乘运算：
我们先定义α = θ/2。 n为单位向量。
q = [cosα   nsinα] = [cosα   xsinα   ysinα   zsinα]

q 的对数：log q = log([cosα   nsinα]) ≡ [ 0  αn ]

q 的指数：exp p = exp([ 0 αn ]) = [cosα  n sinα]

根据定义，exp p 总是返回单位四元数。

与标量类似，四元数指数运算为对数运算的逆运算： exp(log q) = q

四元数能与一个标量相乘：k q = k[w  v ] = [kw   kv ] = k[w  (x  y  z)] = [kw  kx  ky  kz]
		
####四元数求幂：
已知对一个非零标量a，a^0 = 1， a^1 = a，当t从0变到1时，a^t从1到a。四元数求幂有类似的结论：当t从0变到1， q^t从[1, **0**]到 q 。

四元数 q 代表一个角位移，现在想要得到代表1/3这个角位移的四元数，可以这样计算： q^1/3。 q^2代表的角位移是 q 的两倍。

假设q代表绕x轴顺时针旋转30度，那么q^2代表绕x轴顺时针旋转60度， q^-1/3代表绕x轴逆时针旋转10度。

注意，四元数表达角位移时使用**最短圆弧**，不能"绕圈"。比如，对上述例子，q^4不是预期的绕x轴顺时针旋转240度，而是逆时针80度。所以(q^4)^1/2不是 q^2，一般，指数运算的代数公式，如(a^s)^t = a^(st)，对四元数都不适用。

公式：q^t = exp( t log q )

####四元数插值 -- "slerp"：
slerp是球面线性插值的缩写（Spherical Linear Interpolation）。它可以在两个四元数间平滑插值。避免了欧拉角插值的所有问题。
slerp有三个操作数。首先是两个四元数，将在它们两个间插值。开始和结束四元数设为q0和q1。插值参数设为t，t在0到1之间变化。slerp函数：slerp( q0, q1, t)，将返回 q0到 q1之间的插值方位。

理论上四元数插值的运算：
![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-23.png)	

也可以用此公式：	
![]({{ site.url }}/images/post/2015-09-10/Euler-Quaternion-24.png)	

ω是两个四元数的夹角，可以用四元数的点乘得到ω的cos值。


---
> 参考：
《3D数学基础：图形游戏开发》