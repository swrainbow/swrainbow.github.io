---
title: Games101-着色（光照与基本着色模型)
date: 2023-07-05 17:27:28
tags: [图形学， Shading]
categories: [games101 系列]
cover: https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230718210215.png
---
# 着色之前
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705172811.png)

模型变换 ——视图变换——投影变换——视口变换——光栅化

把这些三角形画在屏幕上后，这些像素的颜色是什么呢？这就是着色的工作。
# 着色
## 定义
着色的定义：物体应用材质的过程

我们为什么能看到物体：人眼接收到了物体来的光。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705172853.png)

- 高光
- 漫反射
- 环境光照

## Shading point
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705172938.png)

- 观测方向$\hat{v}$
- 法线$\hat{n}$
- 光照方向$\hat{l}$
- 表面参数(颜色，shininess)

着色局部性：只考虑这个点，不考虑其他物体的存在（不考虑阴影）。

## 泛光反射
泛光反射即环境光，这里做出假设：任何一个点接受环境光相同。
可以得到理想经验模型：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705173034.png)

其中$k_{a}$表示表面对环境光的反射率，$I_{a}$表示入射环境光的亮度。
这个式子和$\hat{l},\hat{n},\hat{v}$都没关系，可以证实这个理想模型$L_{a}$是常数即一个颜色。


## Lambert漫反射模型
漫反射使物体有体积感。

漫反射是光从一定角度入射之后从入射点向四面八方反射，且每个不同方向反射的光的强度相等。
产生漫反射的原因是物体表面的粗糙，导致了这种物理现象的发生。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705173110.png)

两个理论：
- Lambert余弦定理：$\hat{l}$和$\hat{n}$夹角决定光照强度，应该将光强乘上$cos\theta = l*n$。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705173138.png)

- 能量衰退定理：点光源，半径越大，能量越小。应该将光强除上$r^{2}$(r为光源到入射点距离)(后续光线追踪有物理解释)
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705173227.png)

根据这两个理论，可以得到Lambert漫反射模型：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705173252.png)
其中$k_{d}$为漫反射系数，修改$k_{d}$可以得到物体表面不同的颜色，$I$为入射光强，$r$为光源到入射点距离，$n,l$分别是法向量和入射方向，$max$是为了剔除夹角大于90°的光（负数）。
漫反射系数$k_{d}$：表示为一个vector→RGB三通道→定义一个颜色
观察这个式子，其中没有出现$\hat{v}$，也证实了漫反射与观察方向无关。
Lambert漫反射模型只是一个经验模型，与实际物理有差异。

经过漫反射效果如下：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705173318.png)

## 高光
### Phong反射模型

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705173349.png)

其中$\hat{R}$为镜面反射方向。

产生高光的条件：
- 光滑的表面
- 观察方向接近镜面反射方向（$\hat{v}$和$\hat{R}$足够接近才能看到）

如何判断$\hat{v}$和$\hat{R}$的距离：$cos\alpha=\hat{v}\cdot\hat{R}$

于是得到Phong反射模型：
$L_{s} = k_{s}(I/r^{2})max(0,cos\alpha)^{p} = k_{s}(I/r^{2})max(0,cos(\hat{v}\cdot\hat{R}))^{p}$

其中$k_{s}$为镜面反射系数，经验模型简化忽略能量损失，$I$为入射光强，$r$为光源到入射点距离，$max$用来提出大于90°的光（负数）。
- 指数p的原因：防止反射光过大，离反射光越远就越不应该看见反射光，需要一个指数p加速衰减，如下图所示
- 指数p的作用：控制高光的大小，一般用100~200


![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705173422.png)

然而，Phong模型有一个缺点：不方便计算$cos(\hat{v}\cdot\hat{R})$的值，于是有了Blinn-Phong反射模型。

### Blinn-Phong反射模型

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705173534.png)

核心思想：引入半程向量$\hat{n}$，即当观察方向$\hat{v}$和镜面反射方向$\hat{R}$接近时，认为半程向量$\hat{h}$与法线方向$\hat{n}$接近。
其中

$\hat{h}=bisector(\hat{v},\hat{l})\\
=\frac{\hat{v}+\hat{l}}{||\hat{v}+\hat{l}||}$

于是得到Blinn-Phong反射模型：
$L_{s}=k_{s}(l/r^{2})max(0, \hat{n}\cdot \hat{h})^{p}$


## Blinn-Phong整体计算公式
最终，得到着色整体计算公式。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705173644.png)

