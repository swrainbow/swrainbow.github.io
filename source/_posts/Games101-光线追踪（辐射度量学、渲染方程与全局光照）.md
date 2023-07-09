---
title: Games101-光线追踪（辐射度量学、渲染方程与全局光照）
date: 2023-07-06 17:13:47
tags: [图形学]
categories: [games101 系列]
---

# 辐射度量学
辐射度量学可以提供更符合物理规律的量来描述光线。
并能解决：
- whitted-style光追并没有对漫反射的光线进行追踪，而是直接返回当前着色点颜色。
- Blinn-Phong模型本身就是一个不准确的经验模型，使用的这种模型的whited-style光线追踪自身自然也是不正确的

辐射度量学是对光照的一套测量系统和单位，它能够准确的描述光线的物理性质。
- Radiant Energy（辐射能量）
- Radiant flux（辐射通量）
- Radiant intensity（辐射强度）
- irradiance
- radiance

## Radiant Energy（辐射能量）
$\large
Q[J=Joule]$

辐射能量是指辐射出来的电磁能量。类比于做功。

## Radiant flux（辐射通量）
Radiant flux (power) is the energy emitted, reflected, transmitted or received, per unit time.
 
$\large
\phi\equiv\frac{dQ}{dt}[W=Watt][lm=lumen]^{*}$

辐射通量或者说辐射功率，是指单位时间的能量。类比于功率。

例如用Radiant flux来描述和比较电灯泡的亮度。

## Radiant intensity（辐射强度）
The radiant (luminous) intensity is the power per unit solid angle(立体角) emitted by a point light source.
 
$\large
I(w)\equiv\frac{d\phi}{d\omega}\\
\large
[\frac{W}{st}][\frac{lm}{sr}=cd=candela]$

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706171538.png)

辐射强度是指单位立方角上的辐射通量(power)。
solid angle（立方角）：通过类比二维弧度得到
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706171609.png)

微分立体角$d\omega$的计算：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706171629.png)

首先建立球面坐标系，则$rd\theta$就是微分面积元的长，$rsin\theta d\phi$就是微分面积元的宽。

验证球的立体角是$4\pi$：

$\large
\Omega=\int_{S^{2}}d\omega \\
=\int_{0}^{2\pi}\int_{0}^{\pi}sin\theta d\theta d\phi\\
=4\pi$

各向同性点光源计算Radiant intensity：

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706171656.png)

## Irradiance(Power)
The irradiance is the power per unit area incident on a surface point.
 
$\large
E(x)\equiv\frac{d\phi(x)}{dA}\\
[\frac{W}{m^{2}}][\frac{lm}{m^{2}}=lux]$

iraadiance是指单位照射面积所接收到的power。

照射面积的计算：

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706171719.png)

这也解释了Lambert漫反射乘以$cos\theta$的原因。

能量衰减：

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706171738.png)

## Radiance
The radiance (luminance) is the power emitted, reflected, transmitted or received by a surface, per unit solid angle, per projected unit area。
 
$\large
L(p,\omega)\equiv\frac{d^{2}\phi(p,\omega)}{d\omega dAcos\theta}\\
[\frac{W}{srm^{2}}][\frac{cd}{m^{2}}=\frac{lm}{srm^{2}}=nit]$

Radiance是指每单位立体角，每单位垂直面积的功率。
同时指定了光的方向与照射表面所受到的亮度。

垂直面积：
在irradiance中，定义的是每单位照射面积，而在radiance当中，为了更好的使其成为描述一条光线传播中的亮度，且在传播过程当中大小不随方向改变，所以在定义中关于接收面积的部分是每单位垂直面积，而这一点的不同也正解释了图中式子分母上的$cos\theta$。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706171829.png)

图中$dA$是irradiance中定义的，而$dA^{\perp}$才是radiance中定义的面积。

$\large
dA^{\perp}=dAcos\theta$

radiance和irradiance的关系：

$\large
L(p,\omega)=\frac{dE(p)}{d\omega cos\theta}$


![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706171853.png)

观察一下积分后的式子，$E(p)$就是点p的irradiance，其物理含义是上文所提到过的点p上每单位照射面积的功率，而$L_{i}(p,\omega)$指入射光每立体角，每垂直面积的功率，因此积分式子右边的$cos\theta$解释了面积上定义的差异，而对$d\omega$积分，则是相当于对所有不同角度的入射光线做一个求和，那么该积分式子的物理含义便是，一个点(微分面积元)所接收到的亮度(irradiance)，由所有不同方向的入射光线亮度(radiance)共同贡献得到。

# 双向反射分布函数(BRDF)
BRDF理解光线的角度：一个微分面积元在接收到一定方向$\omega_{i}$上的亮度($dE(W_{i})$)之后，再向不同(any other)方向$\omega$把能量辐射出去($dL_{r}(w_{r})$)。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706171935.png)

BRDF就是描述从不同方向入射之后(from each incoming direction)，朝不同方向反射(into each outgoing direction $\omega_{r}$)分布情况的函数。
具体来说，BRDF为朝某个方向发出反射光radiance与入射光irrandiance的比值。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706171950.png)

BRDF函数接受两个参数，入射方向$\omega_{i}$和出射方向$\omega_{r}$，函数值为反射光的radiance与入射光irradiance的比值。
# 反射方程
反射方程：

$f(x):输入光源的radiance\rightarrow 输出反射的radiance$

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706172017.png)

反射方程只考虑一个光源直接引起的反射，不考虑其他物体引起的间接反射。

前面提到，radiance:$L(p,\omega)$是描述光最重要的物理量且光线有线性性质，所以相机受到的$w_{r}$方向反射光的radiance:$L_{r}(p,\omega_{r})$应该是所有方向入射光$w_{i}$贡献radiance的积分。

$w_{i}$方向入射光radiance为$L_{i}(p,\omega_{i})$，由$L(p,\omega)=\frac{dE(p)}{d\omega cos\theta}$，$w_{i}$方向入射光irrdiance为$L_{i}(p,w_{i})cos\theta_{i}d\omega_{i}$。

而BDRF描述入射光朝不同方向反射的分布情况，即$BRDF=\frac{radiance_{w_{r}}}{irrdiance_{w_{i}}}$，所以一个入射方向$w_{i}$朝反射方向$w_{r}$贡献的radiance为$BRDF\times irrdiance_{w_{i}}$。

那么朝反射方向$w_{r}$的radiance: $L_{r}(p,\omega_{r})$：

$L_{r}(p,\omega_{r})=\sum{radiance_{w_{i}\rightarrow w_{r}}}\\
=\int_{H^{2}}BRDF\times irrdiance_{w_{i}}\\
=\int_{H^{2}}f_{r}(p,\omega_{i}\rightarrow\omega_{r})L_{i}(p,\omega_{i})cos\theta_{i}d\omega_{i}$

# 渲染方程
## 渲染方程式子

前面提到，入射光线的radiance不仅仅是光源所引起的，还有可能是其他物体上着色点的反射光线的radiance，恰好反射到当前的着色点p(即间接光照)，同时其他物体上的反射光线的radiance依然也是由直接光照和间接光照构成，因此这与whitted-style当中的光线追踪过程十分类似，也是一个递归的过程。所以说想要解这样一个方程还是比较难的。
渲染方程在反射方程的基础上添加了一个自发光项(Emission term)：
$L_{o}(p,\omega_{o})=L_{e}(p,\omega_{o})+\int_{\Omega^{+}}{L_{i}(p,\omega_{i})f_{r}(p,\omega_{i},\omega_{o})(n\cdot w_{i})}d\omega_{i}$

## 理解渲染方程
理解渲染方程，就是在上面对反射方程的理解基础上加上自发光。
一个点光源和单个物体的场景：

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706172140.png)
多个点光源和单个物体的场景
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706172149.png)
面光源：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706172215.png)
考虑其他物体的间接光照：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706172232.png)

如上图所示，把其他物体同样考虑成面光源， 对其所占立体角进行积分即可，只不过对其它物体的立体角积分不像是面光源所有入射方向都有radiance，物体的立体角可能只有个别几个方向有入射的radiance(即多次物体间光线反射之后恰好照射到着色点x)，其它方向没有，但本质上都可以视作是面光源。
## Deriving the Rendering Equation(渲染方程的派生)
在这个式子中，未知量反射radiance在方程左侧，入射光radiance在积分内。这种方程在数学上称作Fredholm Integral Equation of the second kind(弗雷德霍姆第二定理)并且有以下规范形式：

$\large
l(u)=e(u)+\int{l(v)K(u,v)dv}$

其中，L是未知量，e是已知量，K是kernel of the integral equation(积分方程的核)
（也就是通过这个定理可以进行化简 具体解法我也没搞明白）

## 化简并求解渲染方程
线性操作
In fact, one can think of a real-valued function as an infinite-dimensional vector where each “element” gives the value of the function when it is evaluated at a particular point. 如果将函数看作无限维向量，其中向量将每个点变换到这个点对应的函数值。
用$(M\circ f)u$表示对函数$f(u)$进行$M$的线性变换。
则

$h(u)=(M\circ f)(u)$

其中$M$是一个线性变换，$f$和$u$是自变量u的线性函数。
就像向量的线性操作法则，对于函数的线性操作法则同样适用：

$\large
M\circ(af+bg)=a(M\circ f)+b(M\circ g)\\
I\circ f = f$

所以，很多线性函数可以表示为对函数的线性操作：

$\large
\int{k(x,x')f(x')dx'}\equiv (K\circ f)(x)\\
\frac{\partial f}{\partial x}(x)\equiv (D\circ f)(x)
$

Ref：

- https://inst.eecs.berkeley.edu/~cs294-13/fa09/lectures/scribe-lecture3.pdf
- https://cgg.mff.cuni.cz/~jaroslav/teaching/2015-npgr010/slides/07 - npgr010-2015 - rendering equation.pptx
## 线性求解渲染方程
根据线性操作的知识，可以将方程写做线性操作形式：

$\large
L=E+K\circ L$

下面进行线性求解：

$
L=E+KL\\
IL-KL=E\\
(I-K)L=E\\
L=(I-K)^{-1}E$

其中$I$为单位矩阵，再利用二项式定理化简：

$\large
L=(I+K+K^{2}+K^{3}+\ldots)\\
L=E+KE+K^{2}E+K^{3}E+\ldots
$

其中E为光源所发出来的光，K为反射算子，这个式子物理含义如下：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706172358.png)
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706172411.png)
发现：考虑次数越多越接近真实图片效果,趋近收敛






