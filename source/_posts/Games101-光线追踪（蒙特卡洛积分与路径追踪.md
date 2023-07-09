---
title: Games1-101 光线追踪（蒙特卡洛积分与路径追踪
date: 2023-07-06 17:44:27
tags:
---
# 蒙特卡洛积分
概率论前置：

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706174606.png)

- pdf(x)/p(x): 概率密度函数
- E[x]: 期望

蒙特卡洛积分是一种求定积分值的方法：对函数值进行多次采样求均值作为积分值的近似。
蒙特卡洛需要知道xf(x)，xp(x)的映射关系，从而对积分值进行近似：

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706174629.png)

蒙特卡洛积分的性质：

- 越多采样，越小误差
- 采样对象和积分对象一一对应

下面以$p(x)=C(常数)$为例，即均匀采样。
例如对于一个函数$f(x)$，它的图像如下所示：

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706174648.png)

从几何角度：我们知道，求解一个定积分等于函数图像在定义域与x轴围成的面积，即上图阴影部分面积。

黎曼积分角度：对于每极小$\Delta x_{i}$，取$\Delta x_{i}$作底，$f(\Delta x_{i})$作高的极小长方形，面积为$\Delta s_{i}$，则面积即定积分值是$\sum\Delta s_{i}$
蒙特卡洛积分：对于一个$x_{i}$，每次采样用函数值$f(x_{i})$作高，每次采样用函数值$f(x_{i})$作高，|ab|作宽长方形近似面积$s_{i}$，N次采样的长方形求平均为$\large \frac{\sum s_{i}}{N}$定积分的值。
下面从公式验证均匀采样蒙特卡洛积分的几何角度：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706174706.png)
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706174717.png)

# 蒙特卡洛路径追踪 (Monte Carlo Path Tracing)
Ray Tracing是Whited Style下的问题，有以下两点：

- 总是镜面反射
- 不进行漫反射

而蒙特卡洛路径追踪就是解决这些问题，从而得到更真实的光照渲染。

下面按照这个逻辑图进行求解：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706174755.png)


渲染方程：

$\large
L_{o}(p,\omega_{o})=L_{e}(p,\omega_{o})+\int_{\Omega^{+}}{L_{i}(p,\omega_{i})f_{r}(p,\omega_{i},\omega_{o})(n\cdot w_{i})}d\omega_{i}$

求解渲染方程需要解决两个问题：

- 积分的计算
- 递归形式（前面提到将其他物体间接反射的光也当作面光源的直接光照进行计算）

下面讨论将渲染方程等同反射方程：

$\large
L_{o}(p,\omega_{o})=\int_{\Omega^{+}}{L_{i}(p,\omega_{i})f_{r}(p,\omega_{i},\omega_{o})(n\cdot w_{i})}d\omega_{i}$

一、
首先只考虑直接光照渲染一个pixel：

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706174859.png)

注意，我们认为反射中的方向向量统一朝外。

$L_{o}(p,\omega_{o})=\int_{\Omega^{+}}{L_{i}(p,\omega_{i})f_{r}(p,\omega_{i},\omega_{o})(n\cdot w_{i})}d\omega_{i}$

观察反射方程，只有一个定积分，其物理含义就是着色点p到眼睛的radiance。
所以采用蒙特卡洛积分。
因为是对半球上对不同方向进行采样，且漫反射与方向无关，均匀分布。
所以可以认为这是均匀采样。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706174921.png)

$p(\omega_{i})=\frac{1}{2\pi}$

运用蒙特卡洛积分得到：

$
\omega_{i}\sim p(\omega_{i})=\frac{1}{2\pi}\\
L_{o}(p,\omega_{o})\approx\frac{1}{N}\sum\limits_{i=1}^{N}\frac{L_{i}(p,\omega_{i})f_{r}(p,\omega_{i},\omega_{o})(n\cdot \omega_{i})}{p(\omega_{i})}
$

伪代码如下：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706175015.png)

二、
下面考虑其他物体的间接光照：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706175031.png)

我们将其他物体的经过能量损耗的光照也当作直接光源，进行递归即可。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706175052.png)
至此，就用蒙特卡洛积分解出了方程的积分值，也通过考虑直接光照与间接光照解决了递归的问题。

但这样做也会有缺陷：
问题一、
光线数量爆炸
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706175109.png)
若采样的光线数为N，光线弹射次数为#bounces，则最终眼睛里有$N^{\#bounces}$条光线。这样的计算量是无法负担的。解决方法也很简单，当N=1时，函数将不再指数级增长。也就是说，我们每次只随机采样一个方向的光线。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706175130.png)

不过这样也会带来问题，过于noisy。因为蒙特卡洛实际上是对积分进行近似，采样越多近似越精准，采样一条路径结果肯定有失偏颇。解决方法是重复多次寻找多条路径，再将多条路径的结果求平均。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706175146.png)
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706175157.png)

ray_generation: 进行N次光线采样并平均；shade：发射一条光线

问题二、
shade递归函数出口
方法1：限制弹射次数 不符合实际物理
方法2：俄罗斯轮盘赌

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706175216.png)

给你一把左轮，两发子弹，你不知道哪一发会真正的射出子弹，因此拿这把左轮射自己，你有4/6的概率活下来，这就是俄罗斯轮盘赌的概念。
应用在路径追踪：假设概率P，有P的概率光线会继续追踪并返回着色结果为$L_{o}/P$，否则1-P的概率停止递归并返回着色结果0。这样保证返回的Radiance期望始终为$L_{o}$。证明如下：

$E=P*(L_{o}/P)+(1-P)*0=L_{o}$

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706175233.png)

此时，算法本身已经正确。但效率不高，如图所示：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706175249.png)

在每次计算直接光照的时候，通过均匀采样任选一个方向，但很少会的光线可以hit光源，尤其当光源较小的时候，这种现象越明显，大量采样的光线都被浪费了。

因此我们将采样的对象由角度变为光源(的面积)，这样所有采样的光线都一定会击中光源(如果中间没有别的物体)，没有光线再会被浪费了。假设光源的面积为A，那么：

$\large
\int pdf~dA=1\\
pdf=\frac{1}{A}
$

观察渲染方程(反射方程)：

$\large
L_{o}(p,\omega_{o})=\int_{\Omega^{+}}{L_{i}(p,\omega_{i})f_{r}(p,\omega_{i},\omega_{o})(n\cdot w_{i})}d\omega_{i}$

其中积分对象为$d\omega_{i}$，前面提到蒙特卡洛要求采样和积分对象一一对应。所以这里需要运用换元法将$d\omega_{i}$换元为$dA$。
首先找到$\omega$和$A$的关系如图：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706175328.png)
找到关系后，进行换元。

$
L_{o}(p,\omega_{o})=\int_{\Omega^{+}}{L_{i}(p,\omega_{i})f_{r}(p,\omega_{i},\omega_{o})cos\theta}d\omega_{i}\\
=\int_{\Omega^{+}}{L_{i}(p,\omega_{i})f_{r}(p,\omega_{i},\omega_{o})\frac{cos\theta cos\theta'}{||x'-x||^{2}}}dA$

这样就可以利用蒙特卡洛积分对光源进行采样并求解积分值。对于间接光照，依然采用先前的方法进行光线方向的均匀采样。最终伪代码如下，分直接光照和间接光照两部分计算：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706175421.png)

tips:计算直接光照的时候还需要判断光源与着色点之间是否有物体遮挡，该做法也很简单，只需从着色点x向光源采样点x’发出一条检测光线判断是否与光源之外的物体相交即可，如图所示:
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706175436.png)


