---
title: unity-自定义渲染管线-(srp-01)
date: 2023-07-17 03:04:30
tags: [unity]
categories: [unity-catlike-coding(SRP)]
cover: https://strainbow.oss-cn-hangzhou.aliyuncs.com/self-transparent.gif
---
unity-catlike-coding教程 [原文链接](https://catlikecoding.com/unity/tutorials/custom-srp/custom-render-pipeline/)

# 前言
”渲染管线”是指GPU硬件层面的渲染管线，而“自定义渲染管线”中的“渲染管线”是应用层面的高级渲染管线，是需要通过api调用来搭建的管线。

渲染管线是游戏引擎完成一帧画面的高层渲染逻辑和流程。
## GPU渲染流水线
可参考之前发过的文章
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230708192233.png)

## SRP

自定义渲染管线就是这样的一个东西：我可以自定义渲染管线中的任何流程，并且已经给我准备好了基础的工具，不需要像编写OpenGL一样自己编写所有的操作。

本章讲如何在SRP下创建一个自己的渲染管线。
Unity规定创建自定渲染管线必须包括两部分：渲染管线资源，渲染管线实例，其中渲染管线资源就像是渲染管线实例的工厂类。
什么是渲染管线？它决定CPU传过来的数据在GPU中绘制在哪，什么时候绘制，怎样绘制。
然而，GPU不能像CPU一样丢一个函数让它执行绘制，我们只能将数据和绘制通过命令的方式塞给GPU，GPU再依次执行这些命令。

## 代码层级

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230717030516.png)

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230717030529.png)

# CommandBuffer&ScriptableContext

以下简称CommandBuff为buff，ScriptableRenderContext为context。
GPU是一个独立的计算设备，如果想要它进行绘制，只能将各种数据经过buffer塞给它。而命令提交分为三步：准备命令，提交命令，执行命令。
提交命令指的是将context列表中的命令通过context.Submit函数提交给GPU，GPU再按照context列表中的顺序依次执行这些命令。
准备命令分为两种，一种是通过buffer函数添加命令，一种是通过context的函数添加向context的队尾添加命令。buffer也可以认为是存储命令的列表，但这些命令不能直接提交给GPU，必须通过context.ExecuteCommandBuffer函数将buffer中有的命令按顺序复制到context队尾中。
buffer中的命令一般是context的前置准备，在buffer中提交好context命令所需要的准备后，将它们execute到context中，最后通过context.Submit一并提交给GPU。
context是RenderPipeline::Render重载函数的参数，每个渲染管线只有一个，放整个渲染管线想让GPU执行的命令，submit后context清空。buffer是自定义变量，可以有多个，就是存放命令的缓存，execute后命令还存在在buffer中。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230717030710.png)


# DrawCalls
什么是Draw Call？

1，Draw Call这个命令的发起方是CPU，接收方是GPU；2，Draw Call中传递的信息为“需要渲染的图元列表”。而图元列表其实就是一系列顶点、材质、纹理、着色器等数据。

Draw Call是CPU调用图像编程接口，如OpenGL中地glDrawElements命令，以命令GPU进行渲染的操作

从CPU为起点，到Draw Call调用的流程。参考《Shader入门精要》，其经历了如下过程：1，把数据加载到显存中，把渲染所需的所有数据（顶点、法线、纹理坐标等）从硬盘加载到RAM，再从RAM加载到显存；2，设置渲染状态，设置着色器、光源属性、材质等；3，调用Draw Call，告诉GPU开始渲染。

在每次调用Draw Call之前CPU都要向GPU发送许多内容（包括数据、状态、指令等），因此Draw Call的增多会使CPU压力过大，造成性能瓶颈。而为了减少Draw Call，我们就引入了批处理（Batching） 的方法，把多个小量Draw Call合并成一个大的Draw Call。

由此我们知道了何为Draw Call，Command Buffer的作用以及为什么我们要减少Draw Call。在这一章中，我们就会编写一系列Shader，使其支持SRP Batcher、GPU Instancing和Dynamic Batching这些批处理技术

## unity中的着色器

单的Shader对象可能只包括一个通道，但更复杂的着色器可以包含多个通道。我们举一个很简单的例子就可以打破“Pass就是一套通俗意义上Shader”的思想。我们举例一个所谓的“简单的只包括一个Pass”的Shader对象，一个UnlitShader，在这个Pass中，我们让物体绘制成纯红色。简单思考一下，这个Pass里的vertex和fragment函数很简单吧，并且只要一个Pass就够了吧。那我们再举例一个“更复杂”的Shader对象，在这个SubShader中，我们还是把物体绘制成纯红色，但是我们还希望这个物体能投射阴影。这时候再思考一下，在这个Shader中，我们总共需要一个Pass来绘制纯红色，还需要一个Pass来让物体渲染在阴影贴图上，意味着在这个SubShader中我们需要定义2个Pass。通过这样一个例子（纯色渲染、纯色渲染+投射阴影），我们就能区分一个SubShader和一个Pass的区别，SubShader决定了我们在整个渲染管线流程中渲染这个物体的所有行为（一个物体可能被渲染多次，可能一次屏幕上，一次阴影贴图上），而一个Pass决定了我们对这个物体的一次渲染行为。而在理解这点后，我们就会发现，如何理解SubShader和Pass就取决于我们对通俗意义上的Shader的理解，如果认为Shader确定了一次对物体的渲染行为（走一个顶点着色器，再走一个片元着色器），那Pass就是Shader；如果认为Shader确定了在整套渲染流程中物

## 无光照着色器 Unlit Shader

```glsl
Shader "Custom RP/Unlit"
{
    Properties {}

    SubShader
    {
        Pass {}
    }
}
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230717184105.png)
一个材质都需要其对应的Shader（也就是说材质的创建依赖于Shader）。在我看来，Shader就好比定义一个C#类，而材质就是这个类的实例，在Shader中我们往往会定义一些参数，而在材质中我们需要赋予和确定这些参数的值。

Render Queue代表了此材质的渲染队列，简单来说，Render Queue意味着使用这个材质的物体在渲染管线中被渲染的顺序。官方文档中说到，Render Queue的值应该处于[0,5000]，或者为-1使用着色器的渲染队列。

Render Queue的值越大，渲染越靠后。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230717184226.png)

## 2 批处理 Batching

任何一次Draw Call都需要CPU和GPU之间的通信，如果有大量的数据需要从CPU传递到GPU，那么就会造成GPU闲置的情况（渲染完当前任务后等待CPU传递下一帧数据）。同时，CPU在传递数据的时候是不能做其他事情的。这两个问题都会造成帧率下降。在目前，我们的管线渲染的方法是这样的：每一个物体对应一个Draw Call。这样的方法显然是低效且不聪明的，一旦我们数据量大起来，性能是吃不消的

批处理的思路就在于：归纳出不同数据之间的特性（相似性），将相似（或者冗余）的数据合并或者一次传递给GPU

如下图，我在一个场景中拜访了3种颜色的小球，一共17个小球，从Stats界面中，我们可以看到进行了18次Batches（即18次Draw Call，多的一次为天空盒的绘制），每次Draw Call传递一个小球的所有数据，显然这是不聪明的
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230717185210.png)

## 2.1 SRP批处理 SRP Batcher

发生一次Draw Call之前，CPU需要做两件事：1，把数据加载到显存，即把网格顶点、法线、纹理等加载到显存中；2，设置渲染状态，包括定义网格使用哪个纹理、哪个材质，还有光源属性等。

那么我们理解了，GPU想要进行一次渲染操作（Draw Call)，粗略看来需要两类信息：1，Mesh顶点；2，材质信息。（实际肯定更复杂）

CPU会在Draw Call前进行一次设置渲染状态。设置渲染状态流程：CPU首先会判断当前GPU中当前渲染状态是否为需要的渲染状态，如果是，则不用更改渲染状态，完成设置；如果不是，则设置为新的渲染状态，也叫做更换渲染状态，而更换渲染状态也就叫做SetPass Call。对于SetPass Call我们必须知道的一点是，SetPass Call非常耗时！！

当多个物体的常量结构体的成员完全相同时（bytestride和bytesize），就可以将它们的常量结构体合并到一个大的常量结构体中。在使用数据时，只需要知道这个物体对应在内存的偏移即可。也就是说，传递这几个物体的常量数据时，CPU只需要向GPU传递这一次。在今后的draw call中，只需要传递每个物体的cbuffer在大cbuffer的偏移。

对于一次DrawRenderers，Unity会遍历所有要绘制的物体（Renderer），然后收集所有出现的材质，整理成一个大的CBuffer，一次传递给GPU的常量缓冲区，同时对这些物体的一些信息（Model矩阵等）也整理成一个大CBuffer，一次传递给GPU的常量缓冲区。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230717201457.png)

## 2.2 GPU实例化 GPU Instancing

针对于每个物体自身不同的材质属性，优化Draw Call性能的另一个方法是GPU Instancing，它的作用是将多个使用相同Mesh相同Material的Objects放在一次Draw Call中绘制。其中，不同Object的材质属性可以不同。CPU会收集每个物体的Transform信息和Material Properties然后构建成一个数组发送给GPU。GPU根据数组迭代绘制每个实体。

其主要做的事就包括了每实例数据的定义（数组形式）、每实例唯一ID的构建、每实例数据的获取。

```glsl
#ifndef CUSTOM_UNLIT_PASS_INCLUDED
#define CUSTOM_UNLIT_PASS_INCLUDED

#include "../ShaderLibrary/Common.hlsl"

//使用Core RP Library的CBUFFER宏指令包裹材质属性，让Shader支持SRP Batcher，同时在不支持SRP Batcher的平台自动关闭它。
//CBUFFER_START后要加一个参数，参数表示该C buffer的名字(Unity内置了一些名字，如UnityPerMaterial，UnityPerDraw。
// CBUFFER_START(UnityPerMaterial)
// float4 _BaseColor;
// CBUFFER_END

//为了使用GPU Instancing，每实例数据要构建成数组,使用UNITY_INSTANCING_BUFFER_START(END)来包裹每实例数据
UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
    //_BaseColor在数组中的定义格式
    UNITY_DEFINE_INSTANCED_PROP(float4,_BaseColor)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)

//使用结构体定义顶点着色器的输入，一个是为了代码更整洁，一个是为了支持GPU Instancing（获取object的index）
struct Attributes
{
    float3 positionOS:POSITION;
    //定义GPU Instancing使用的每个实例的ID，告诉GPU当前绘制的是哪个Object
    UNITY_VERTEX_INPUT_INSTANCE_ID
};

//为了在片元着色器中获取实例ID，给顶点着色器的输出（即片元着色器的输入）也定义一个结构体
//命名为Varings是因为它包含的数据可以在同一三角形的片段之间变化
struct Varyings
{
    float4 positionCS:SV_POSITION;
    //定义每一个片元对应的object的唯一ID
    UNITY_VERTEX_INPUT_INSTANCE_ID
};

Varyings UnlitPassVertex(Attributes input)
{
    Varyings output;
    //从input中提取实例的ID并将其存储在其他实例化宏所依赖的全局静态变量中
    UNITY_SETUP_INSTANCE_ID(input);
    //将实例ID传递给output
    UNITY_TRANSFER_INSTANCE_ID(input,output);
    float3 positionWS = TransformObjectToWorld(input.positionOS);
    output.positionCS = TransformWorldToHClip(positionWS);
    return output;
}

float4 UnlitPassFragment(Varyings input) : SV_TARGET
{
    //从input中提取实例的ID并将其存储在其他实例化宏所依赖的全局静态变量中
    UNITY_SETUP_INSTANCE_ID(input);
    //通过UNITY_ACCESS_INSTANCED_PROP获取每实例数据
    return UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _BaseColor);
}

#endif
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230717210632.png)

并没有只执行一次Draw Call，而是执行了3次，第一次绘制了511个，第二次绘制了511个，第三次绘制了1个

我的mac GPU的常量缓冲区大小大约就是40KB+Mesh数据+材质部分数据，估计总的大小也就是64KB吧。

## 2.3 动态批处理 Dynamic Batching

第三个优化Draw Call的方法——动态批处理Dynamic Batching。这是一个比较古老的技术，它把多个共享相同材质的小Mesh合并成一个大的Mesh进行绘制。另外，它不支持每Ojbect的材质属性(MaterialPropertyBlock)

GPU Instancing比Dynamic Batching效率更高。同时，Dynamic Batching有一些注意事项，比如涉及不同Scale的网格合并时，不能保证较大网格的法向量是单位长度的（原因未知），同时，绘制顺序会发生变化

## 3. 透明 Transparency
不透明物体和透明物体的渲染的主要区别在于是否完全覆盖原像素的颜色。我们可以使用source和destination的混合模式，source指将要被绘制的颜色，destination指当前像素的颜色。

我们首先在Properties中定义_SrcBlend和_DstBlend。这两个值本应该是枚举值，但我们使用Float来定义它们，同时增加特性，在Editor下更方便地设置它们。最后在Pass中设置混合模式，使用方括号包裹_SrcBlend和_DstBlend，这是种古老的语法。

Opaque物体的混合模式为Src=One、Dst=Zero，即新颜色会完全覆盖旧颜色，而Transparent物体的混合模式为Src=SrcAlhpa、Dst=OneMinusSrcAlpha，使用新片元的透明度为权值混合两者。这个比较简单，《Shader入门精要》中也对混合模式有较为详细的介绍。

## 3.1 透明度测试实例化 Ball of Alpha-Clipped Spheres
Meshball.cs中Awake部分代码如下。
```c#
private void Awake()
    {

        for (int i = 0; i < matrices.Length; i++)
        {
            //在半径10米的球空间内随机实例小球的位置
            matrices[i] = Matrix4x4.TRS(Random.insideUnitSphere * 10f,
                Quaternion.Euler(Random.value * 360f, Random.value * 360f, Random.value * 360f),
                Vector3.one * Random.Range(0.5f, 1.5f));
            baseColors[i] = new Vector4(Random.value, Random.value, Random.value, Random.Range(0.5f,1f));
        }
    }
```

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/self-transparent.gif)