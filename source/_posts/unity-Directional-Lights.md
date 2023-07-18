---
title: unity-Directional Lights(srp-02)
tags: [unity]
categories: [unity-catlike-coding(SRP)]
cover: https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230718171552.png
---

# Lighting
如果我们想创建一个更具有真实感的场景，我们需要模拟光线和物体表面是如何交互的。因此我们需要实现一个比Unlit Shader更加复杂一些的Shader。

## 1.1 带光照着色器 Lit Shader
首先我们复制UnlitPass.hlsl，再将其重命名为LitPass.hlsl，因为Lit和Unlit的整体框架大致是相同的，无非是在Vertex和Fragment方法、传递的数据上有所改动。

接下来就进入到比较关键的地方了，我们需要现在Lit.shader的Pass里增加"LightMode" = "CustomLit"的Tag（"CustomLit"这个名字是自己取的）。
```glsl
Pass
        {
            //设置Pass Tags，最关键的Tag为"LightMode"
            Tags
            {
                "LightMode" = "CustomLit"
            }
            //设置混合模式
            Blend [_SrcBlend] [_DstBlend]
            ZWrite [_ZWrite]

            HLSLPROGRAM
            //告诉Unity启用_CLIPPING关键字时编译不同版本的Shader
            #pragma shader_feature _CLIPPING
            //这一指令会让Unity生成两个该Shader的变体，一个支持GPU Instancing，另一个不支持。
            #pragma multi_compile_instancing
            #pragma vertex LitPassVertex
            #pragma fragment LitPassFragment
            #include "LitPass.hlsl"
            ENDHLSL
        }
```
1.何为Pass Tags（通道标签）？

根据官方描述，Tag是可以分配给Pass的键值对数据。

2.Pass Tags用来做什么？

Unity使用预定义的标签和值来确定如何以及何时渲染给定的Pass。（题外话，我们还可以使用自定义值创建自定义的Pass Tag，并从C#代码访问它们，这个功能我们目前应该用不上）。最常用的预定义Pass Tag为"LightMode",预定义也就是意味着这个Tag的键（即"LightMode"）是Unity内置的，其用于所有渲染管线。值得一提的是，通常SubShader中我们也会赋予SubShader Tags，但SubShader Tags和Pass Tags的工作方式不同，在Pass中设置SubShader Tag是没有效果的，反之亦然。因此，Pass Tags必须放在Pass中定义，SubShader Tags必须放在SubShader中定义。

3.何为"LightMode"标签？

参考官方文档，"LightMode"是Unity预定义的一个Pass Tag，Unity使用它来确定是否在给定帧期间执行该Pass，在该帧期间Unity何时执行该Pass，以及Unity对输出执行哪些操作。"LightMode"是非常重要的一个Pass Tag，在Unity任何渲染管线中其都会被预定义，但其默认值会随着管线不同而不同（比如Built-in和URP，自带的LightMode值不同）。

在SRP中，我们可以为"LightMode"这一Pass Tag创建自定义值，然后通过配置DrawingSettings结构，可以利用这些自定义值在DrawRenderers期间绘制指定Pass（这个方法是很重要的，相当于我们的画笔，在很多地方我们都可以用这个方法）。另外，在SRP中，我们可以使用SRPDefaultUnlit值来引用没有LightMode标签的通道，这也就意味着如果一个Pass中没有定义LightMode的Pass Tag，Unity会自动将其归为"LightMode"="SRPDefaultUnlit"。（这也就是为什么我们的Unlit.shader没有定义"LightMode",但我们依然能通过"SRPDefaultUnlit"来绘制它们的原因）

## 1.2 法线 Normal Vectors

法线球 太熟悉了， 都不知道画了几次了
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230718000408.png)

## 1.3 表面属性 Surface Properties
```glsl
//定义与光照相关的物体表面属性
//HLSL编译保护机制
#ifndef CUSTOM_SURFACE_INCLUDED
#define CUSTOM_SURFACE_INCLUDED

//物体表面属性，该结构体在片元着色器中被构建
struct Surface
{
    //顶点法线，在这里不明确其坐标空间，因为光照可以在任何空间下计算，在该项目中使用世界空间
    float3 normal;
    //表面颜色
    float3 color;
    //透明度
    float alpha;
};

#endif
```
```glsl
float3 color = GetLighting(surface);
return float4(color, surface.alpha);
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230718001748.png)

# 2 光源 Lights
## 2.1 光源数据结构

```glsl
//用来定义光源属性
#ifndef CUSTOM_LIGHT_INCLUDED
#define CUSTOM_LIGHT_INCLUDED

struct Light
{
    //光源颜色
    float3 color;
    //光源方向：指向光源
    float3 direction;
};

//返回一个配置好的光源，初始化为Color白色，光线从上垂直向下投射（不明确坐标系，但由于教程中在世界空间下计算光照，因此这里多半指的是世界空间）
Light GetDirectionalLight()
{
    Light light;
    light.color = 1.0;
    light.direction = float3(0.0,1.0,0.0);
    return light;
}

#endif
```

## 2.2 光照函数 Lighting Function
```glsl
float3 IncomingLight(Surface surface,Light light)
{
    return saturate(dot(surface.normal,light.direction)) * light.color;
}

//新增的GetLighting方法，传入surface和light，返回真正的光照计算结果，即物体表面最终反射出的RGB光能量
float3 GetLighting(Surface surface,Light light)
{
    return IncomingLight(surface,light) * surface.color;
}

//GetLighting返回光照结果，这个GetLighting只传入一个surface
float3 GetLighting(Surface surface)
{
    //光源从Light.hlsl的GetDirectionalLight获取
    return GetLighting(surface,GetDirectionalLight());
}
```

## 2.3 将光源数据传递给GPU Sending Light Data to the GPU
通过CBUFFER（Constant Buffer）包裹从Cpu发送过来的两个光源属性（即一个color值和一个方向），CBUFFER名为_CustomLight。

我们使用CBUFFER来实现批处理技术SRP Batch，在这里再回忆一下CBuffer的含义。CBuffer，即Constant Buffer，常量缓冲区，用于存放在GPU进行一次Draw Call指令的渲染操作内保持不变的数据，访问速度很快，但内存总量也很小。CPU可以在每次发出Draw Call指令前对常量缓冲区进行修改，在这里也就是每一帧cpu可以修改常量缓冲区中的光源属性（光源是可实时变化的）。

## 2.4 当前起效的光源 Visible Lights
当进行视锥体裁剪（Culling）时，Unity也会找到影响当前可视范围的光源（Unity官方叫做可见光源，但我觉得说是起效的光源更合理）。我们可以使用这一信息，而不是通过RenderSettings.sun这一全局信息。所以，第一步我们要在Lighting.cs中获取到Culling Results，因此在Setup函数中增加一个传入参数CullingResults（并将其存储为Lighting.cs下的一个字段以方便使用）。同时，由于使用CullingResults下的光源信息，我们可以支持多个光源，因此，我们创建并使用SetLights方法代替原来的SetupDirectionalLight。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/light.gif)

## 2.5 Shader目标等级 Shader Target Level
对于累积多个光源的光照计算结果，具有可变次数的循环曾经对于着色器来说是一个难题，虽然现代的GPU可以轻松处理它们，但OpenGL ES 2.0和WebGL 1.0图形API无法处理此类循环。虽然我们可以进行分支控制或者硬编码来避免这类问题，但也会导致生成的着色器代码较为混乱，且性能非常差。因此，在本教程中选择直接屏蔽这些不支持的图形API，我们通过#pragma target 3.5指令将着色器通道的Target Level，从而避免为它们编译OpenGL ES 2.0着色器变体，我们对Unlit和Lit两个Shader都添加该指令。

```glsl
        Pass
        {
            //设置Pass Tags，最关键的Tag为"LightMode"
            Tags
            {
                "LightMode" = "CustomLit"
            }
            //设置混合模式
            Blend [_SrcBlend] [_DstBlend]
            ZWrite [_ZWrite]

            HLSLPROGRAM
            //不生成OpenGL ES 2.0等图形API的着色器变体，其不支持可变次数的循环与线性颜色空间
            #pragma target 3.5
            //告诉Unity启用_CLIPPING关键字时编译不同版本的Shader
            #pragma shader_feature _CLIPPING
            //这一指令会让Unity生成两个该Shader的变体，一个支持GPU Instancing，另一个不支持。
            #pragma multi_compile_instancing
            #pragma vertex LitPassVertex
            #pragma fragment LitPassFragment
            #include "LitPass.hlsl"
            ENDHLSL
        }
```

# 3 BRDF 双向反射分布函数
我们可以通过BRDF（Bidirectional Reflectance Distribution Function）函数来获取更加多变和真实的光照。业界内有许多这样的函数，在本教程中，我们会使用和URP一样的BRDF函数，该函数牺牲了一些画面真实感来换取性能。BRDF函数也是PBR（Physically Based Rendering基于物理的渲染）的核心。对于PBR和BRDF，推荐观看闫老师的《GAMES 101》还有毛星云前辈的PBR白皮书

## 3.1 Incoming Light 入射光

N·L来表示物体表面一块区域（Fragment）接收到入射光的总能量
 ![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230718152729.png)

 ## 3.2 出射光 Outgoing Light
 光打在物体表面后反射出来且刚刚好打到摄像机（或者我们的眼睛）的光
 ![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230718152915.png)

但是当物体表面不完全平坦（实际上完全平坦只是理论上存在，因此实际的物体表面必然是不完全平坦的）时，打在物体表面的光线会被散射（Scattered，即被散射成好几道不同方向的出射光），这是由于物体表面的一块区域是由许多更小的、起伏不平的、方向不同的微表面组成的。这会将入射光分散成多个不同方向的小量出射光（小量是因为能量守恒），而这就会模糊Specular部分，看起来就是物体表面的高光区域变大。但是这些多方向的反射也就导致了另一个好处，就是即使我们不完全对着高光的出射光方向，我们也能看到一些高光部分
 ![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230718152938.png)

 ![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230718153142.png)

 对于光源颜色来说，其定义变为了光垂直照射到完全漫反射的白色物体表面时摄像机观察到的光能量。此外还有其他配置光源的方法，例如可以增加lumen或lux这些光学意义上的参数来模拟更真实的光源，但我们会使用当前的简化PBR模型，即只用光源颜色（实际上考虑光线能量强弱，即Intensity）。

 ## 3.3物体表面属性 Surface Properties
 物体表面可以被完全漫反射，也可以被完全镜面反射（Specular），或者结合两者。我们有许多种方式来控制它们。在这里我们使用Metallic Workflow（金属工作流），这需要我们向Lit.shader的Properties中增加两个物体表面属性。

第一个属性衡量物体表面是金属的还是非金属的。我们将该值控制在[0,1]区间内，1代表物体表面是完全金属的。我们将其默认值设置为完全非金属的，即该值为0。

第二个属性衡量物体表面有多光滑。同样，我们将该值控制在[0,1]区间内，0代表完全不光滑（完全粗糙），1代表完全光滑。我们将其默认值设置为0.5。

将这两个属性也加入名为UnityPerMater的CBUFFER（该CBUFFER段是我们之前用于SRPBatcher的）。接下来，我们在Surface结构体中增加这两个属性。然后在LitPassFragment中构造surface时通过CBUFFER赋值这两个属性。GPU接收端完成了，接下来实现CPU发送端的支持，在PerObjectMaterialProperties.cs中增加对这两个属性的支持。
金属度（Metallic）高的物体表面将几乎所有接收到的光能量以Specular（高光反射）的形式反射出去，同时漫反射量几乎为0。所以我们定义Reflectivity（specular反射率）等于物体表面的metallic属性。同时，我们需要在构造brdf.diffuse时乘以oneMinusReflectivity（遵循能量守恒，Reflectivity值反应Specular占比，而1-Reflectivity值就反应了Diffuse占比）。此时，我们转到Unity中，此时调整物体的_Metallic属性会使物体表面亮度发生变化。_Metallic越高，物体表面越暗，这是因为我们目前只计算了Diffuse部分，Specular部分还没计算

## 3.4 反射率 Reflectivity

金属度（Metallic）高的物体表面将几乎所有接收到的光能量以Specular（高光反射）的形式反射出去，同时漫反射量几乎为0。所以我们定义Reflectivity（specular反射率）等于物体表面的metallic属性。同时，我们需要在构造brdf.diffuse时乘以oneMinusReflectivity（遵循能量守恒，Reflectivity值反应Specular占比，而1-Reflectivity值就反应了Diffuse占比）。此时，我们转到Unity中，此时调整物体的_Metallic属性会使物体表面亮度发生变化。_Metallic越高，物体表面越暗，这是因为我们目前只计算了Diffuse部分，Specular部分还没计算

## 3.5 高光 Specular Color

金属会影响高光的颜色，而非金属不会。我们通过对MIN_REFLECTIVITY(最小高光反射度)和surface.color（物体不吸收的光能量）插值获取到brdf.specular（高光反射占比）。这就意味着，当物体表面的金属度越低，高光能量占比越少，同时高光的颜色也越接近白色；而当物体表面金属度越高时，高光能量占比越多，同时高光的颜色越接近物体表面实际不吸收的光的颜色。

## 3.6 高光强度
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230718170411.png)

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230718171408.png)

# 4 透明材质 Transparency
## 4.1 预乘透明度 Premultiplied Alpha
我们目前要实现的是让diffuse项随alhpa渐变，而specluar项依然维持其值。由于Sorce Blend Mode（透明Unlit材质使用的模式）会将所有计算结果都应用其混合，因此不再适用（不能将Specular单独分离出来）。因此，我们将Source值设置为1，同时Destination依然使用one-minus-source-alpha。

这样，我们的Specular项就会完全保留下来了，但是同时Diffuse项也完全保留下来了。我们通过让brdf的diffuse属性乘以surface.alpha（Premultiplied Alpha）来让Diffuse项依然保留渐变。这种方法就叫做Premultiplied Alpha
```glsl
brdf.diffuse = surface.color * oneMinusReflectivity;
brdf.diffuse *= surface.alpha;
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230718172545.png)

# Shader GUI


