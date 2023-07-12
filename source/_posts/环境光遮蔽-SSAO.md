---
title: 环境光遮蔽-SSAO
date: 2023-07-12 13:38:45
tags:
cover: https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713013827.png
---
参考百人计划课程链接:【技术美术百人计划】图形 4.2 [SSAO算法 屏幕空间环境光遮蔽_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV16q4y1U7S3?p=2&vd_source=dc8bc359da1cacb67fa6054d14862294)

还是应该加上标题序号
# 1. SSAO介绍
## 1.1 关键词介绍
AO:
环境光遮蔽，全称Ambient Occlusion,是计算机图形学中一种着色和渲染技术，模拟光线达到物体的能力的粗略的全局方法，描述光线到达物体表面的能力。

SSAO：
屏幕空间环境光遮蔽，全称Screen Space Ambient Occlusion,一种用于计算机图形中实时实现近似环境光遮蔽效果的渲染技术。它不是场景的预处理，而是一种屏幕后处理技术，通过在屏幕像素的位置布置随机个采样点，然后通过采样点遮蔽项所占的百分比来计算阴影遮蔽强度。

SSAO历史
AO这项技术最早实在Siggraph 2002年会上由ILM(工业光魔)的技术主管Hayden Landis所展示，当时就被叫做Ambient Occlusion。
2007年，Crytek公司发布了一款叫做屏幕空间环境光遮蔽的技术，并用在了他们的看家作孤岛危机上。

## 1.2 流程介绍
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713012311.png)
- 获取深度、法线缓冲。
- 通过深度缓冲计算像素坐标。
- 在着色点法线方向的上半球部分安置随机采样点。
- 对采样点进行遍历，并通过比较采样点的深度和着色点的深度信息对AO进行加权计算。 
- 对获得的AO图进行双边滤波操作，得到模糊图像。
- 对原图像和模糊后的AO图进行合并。

# 二、AO与SSAO计算原理
## 基于法线的半球积分-SSAO
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713012529.png)
上图中黑色表示需要计算的像素样本，蓝色向量表示当前片元的法线向量，白色、灰色为采样点，采样点的多少会影响最后的渲染的效果，其中灰色表示被挡采样点(即遮蔽项)，白色就是正常的采样点，据此判断最终AO的强度。

# 三、SSAO算法实现
## 3.1 Buffer数据的生成设置和获取
c#部分：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713012744.png)
设置相机的参数，生成对应的深度缓冲和法线缓冲。

Shader部分：
声明深度法线缓冲图，这张图同时存储了法线和深度值两个信息。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713012806.png)
法向深度图示例：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713012832.png)

采样获得深度值和法线值
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713012859.png)
对缓冲数据进行解码，其中法线为
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713012917.png)
注意：如果渲染路径设置的是Deferred延迟渲染，法线缓冲可以从G-Buffer中直接获取。

## 3.2 重建相机空间坐标
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713012953.png)
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713013001.png)
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713013011.png)
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713013024.png)
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713013036.png)
最后通过从相机经过对应像素到远平面的射线乘以线性深度值得到最后的像素坐标。

## 3.3 构建法向量的正交基
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713013108.png)
**切线的计算原理**：
因为法线已经确定，所以需要从切平面中随机计算出一条切线。
先随机一个法向半球的向量r。在求得r在法向法向上的投影a，然后再通过r-a得到切线。

## 3.4 AO采样核心
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713013150.png)

# 四、SSAO效果改进
## 4.1 随机正交基
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713013226.png)
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713013234.png)
因为之前的构建TBN矩阵中的随机向量是固定的，所以为了让TBN矩阵具有随机性，那么需要把随机的向量存入纹理中(需要设置为repeat模式)，然后根据平铺来进行随机采样，这样就能得到随机的向量，进而构建随机的TBN矩阵。
## 4.2 AO累加平滑优化
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713013302.png)
_RangeStrength是设置的阈值，如果randomDepth和linearDepth的差值的绝对值大于我们设定的阈值，那么很有可能是非常远(比如天空)，那么就不需要进行AO加权。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713013328.png)
同样的，如果随机点深度值跟自身非常近，那么可能会导致明明再同一平面，但是也会出现AO加权，这个时候设置一个非常小的偏移值来比较判断。只有当前像素深度小于随机点像素深度才能够贡献AO。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713013346.png)
对于权重部分，是随机向量跟半球法线向量相似度越高，所占的权重越小，反之所占权重就为1。应该是跟法线相近的向量一般大概率是没有被遮蔽，没有太大的参考性，所以所占的权重才会小吧。

对于循环内AO的贡献公式为:
ao+=range*selfCheck*weight;

## 4.3 AO模糊
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713013448.png)
跟高斯模糊有些相似，也是通过两个一维卷积核进行卷积运算的，但是不同的是，通过中间像素两边像素的法线的点乘再加上设置的阈值来除去特定频域的信息。
高斯模糊与法线的双边滤波效果可参照下图：
原图：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713013631.png)
高斯模糊：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713013654.png)
基于法线的双边滤波：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713013716.png)

# 五、实现SSAO效果
后续补充。。。。 