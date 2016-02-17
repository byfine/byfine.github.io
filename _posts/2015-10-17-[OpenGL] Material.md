---
layout: post
title: OpenGL 材质 (Material)
description: "OpenGL材质"
modified: 2015-10-17
tags: [OpenGL]
---

## 定义材质：
在真实世界里，每个物体会对光产生不同的反应。每个物体对光的反射强度不一样，对镜面高光也有不同的反应。有些物体不会散射(Scatter)很多光却会反射(Reflect)很多光，结果看起来就有一个较小的高光点(Highlight)，有些物体散射了很多，它们就会产生一个半径更大的高光。

如果我们想要在OpenGL中模拟多种类型的物体，我们必须为每个物体分别定义材质(Material)属性。

当描述物体的时候，我们可以使用3种光照元素：环境光照(Ambient Lighting)、漫反射光照(Diffuse Lighting)、镜面光照(Specular Lighting)定义一个材质颜色。再加上一个镜面高光亮度，这是我们需要的所有材质属性。在片段着色器中，我们创建一个结构体(Struct)，来储存物体的材质属性：

    struct Material
    {
        vec3 ambient;
        vec3 diffuse;
        vec3 specular;
        float shininess;
    };
    uniform Material material;

ambient：定义了在环境光照下这个物体反射的是什么颜色，通常这是和物体颜色相同的颜色。
diffuse：定义了在漫反射光照照下物体的颜色。漫反射颜色被设置为(和环境光照一样)我们需要的物体颜色。
specular：设置的是物体受到的镜面光照的影响的颜色(或者可能是反射一个物体特定的镜面高光颜色)。
shininess：影响镜面高光的散射/半径。

这四个元素定义了一个物体的材质，通过它们我们能够模拟很多真实世界的材质。
这里有一个列表[devernay.free.fr](http://devernay.free.fr/cours/opengl/materials.html)展示了几种材质属性，这些材质属性模拟外部世界的真实材质。

![]({{ site.url }}/images/post/2015-10-17/OpenGL Material 1.png)


## 设置材质：
我们在片段着色器定义了一个uniform的材质结构体，所以我们使用材质的属性来计算光照。
{% highlight C++ %}
void main()
{
    // 环境光
    vec3 ambient = lightColor * material.ambient;

    // 漫反射
    vec3 norm = normalize(Normal); // 标准化法线
    vec3 lightDir = normalize(lightPos - FragPos); //计算并标准化光线向量
    float diff = max(dot(norm, lightDir), 0.0);
    vec3 diffuse = lightColor * (diff * material.diffuse); // 计算漫反射

    // 镜面高光
    vec3 viewDir = normalize(viewPos - FragPos); // 视线向量
    vec3 reflectDir = reflect(-lightDir, norm); // 反射向量，第一个参数要求为光源指向片段的方向，所以要加个负号
    float spec = pow(max(dot(viewDir, reflectDir), 0.0), material.shininess);
    vec3 specular = lightColor * (spec * material.specular);
    
    vec3 result = ambient + diffuse + specular;
    color = vec4(result, 1.0f);
}
{% endhighlight %}

然后在程序中传递uniform的值：

    glUniform3f(glGetUniformLocation(objectShader.Program, "material.ambient"), 1.0f, 0.5f, 0.31f);
    glUniform3f(glGetUniformLocation(objectShader.Program, "material.diffuse"), 1.0f, 0.5f, 0.31f);
    glUniform3f(glGetUniformLocation(objectShader.Program, "material.specular"), 0.5f, 0.5f, 0.5f);
    glUniform1f(glGetUniformLocation(objectShader.Program, "material.shininess"), 32.0f);


## 光的属性：
如果你这样设置运行后，会发现物体非常亮。这是因为环境、漫反射和镜面三个颜色任何一个光源都会去全力反射。而之前们是使用了一个强度值来改变了环境光和镜面反射的强度。
我们定义一个光源的强度向量，来减小环境光的影响：

    vec3 result = vec3(0.1f) * material.ambient;

我们可以用同样的方式影响光源diffuse和specular的强度。

由此，与材质类似，我们同样可以定义光源的属性：

    struct Light
    {
        vec3 position;
        vec3 ambient;
        vec3 diffuse;
        vec3 specular;
    };
    uniform Light light;


一个光源的ambient、diffuse和specular光都有不同的亮度。
ambient：通常设置为一个比较低的亮度，因为我们不希望环境色太过显眼。
diffuse：通常设置为我们希望光所具有的颜色，经常是一个明亮的白色。
specular：通常被设置为vec3(1.0f)类型的全强度发光。
们同样把光的位置添加到结构体中。

然后更新片段着色器：

    vec3 ambient = light.ambient * material.ambient;
    vec3 diffuse = light.diffuse * (diff * material.diffuse);
    vec3 specular = light.specular * (spec * material.specular);


然后设置uniform的光源属性值：

    glUniform3f(glGetUniformLocation(objectShader.Program, "light.ambient"), 0.2f, 0.2f, 0.2f);
    glUniform3f(glGetUniformLocation(objectShader.Program, "light.diffuse"), 0.5f, 0.5f, 0.5f);
    glUniform3f(glGetUniformLocation(objectShader.Program, "light.specular"), 1.0f, 1.0f, 1.0f);


---
> 参考
[LearnOpenGL-CN 材质](http://learnopengl-cn.readthedocs.org/zh/latest/02%20Lighting/03%20Materials/)