---
title: Games101--MVP
date: 2023-07-04 02:14:14
tags: [图形学]
categories: [games101 系列]
---

# MVP变换
MVP变换用来描述视图变换的任务，即将虚拟世界中的三维物体映射（变换）到二维坐标中。

MVP变换分为三步：
- 模型变换(model tranformation)：将模型空间转换到世界空间（找个好的地方，把所有人集合在一起，摆个pose）
-  摄像机变换(view tranformation)：将世界空间转换到观察空间（找到一个放相机的位置，往某一个角度去看）
- 投影变换(projection tranformation)：将观察空间转换到裁剪空间（茄子！）

在这之后，还有一个#视口变换

## 视图变换（View）
视图变换的目的是变换Camera位置到**原点，上方为Y，观察方向为-Z**，即




![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230704185422.png)

$M_{view}=R_{view}T_{view}=\\
\begin{bmatrix}
x_{\hat{g}\times\hat{t}}& y_{\hat{g}\times\hat{t}}& z_{\hat{g}\times\hat{t}}& 0\\  
x_{t}& y_{t}& z_{t}& 0\\
x_{-g}& y_{g}& z_{-g}& 0\\ 
0& 0& 0& 1
\end{bmatrix} \begin{bmatrix}1& 0& 0& -x_{e}\\ 0& 1& 0& -y_{c}\\ 0& 0& 1& -z_{c}\\ 0& 0& 0& 1\end{bmatrix}$


<!-- $\begin{align}
M_{view}&=R_{view}T_{view}\\
&=\begin{bmatrix}
x_{\hat{g}\times\hat{t}}& y_{\hat{g}\times\hat{t}}& z_{\hat{g}\times\hat{t}}& 0\\  
x_{t}& y_{t}& z_{t}& 0\\
x_{-g}& y_{g}& z_{-g}& 0\\ 
0& 0& 0& 1
\end{bmatrix} 
\begin{bmatrix}1& 0& 0& -x_{e}\\ 0& 1& 0& -y_{c}\\ 0& 0& 1& -z_{c}\\ 0& 0& 0& 1\end{bmatrix}
\end{align}$ -->


定义Camera：
- Camera位置$\vec{e}$
- 观察方向$\hat{g}$
- 视点上方向$\hat{t}$

规定：
- Camera的y轴正方向向上，z轴方向是$-\vec{x}\times \vec{y}$（右手系）
- 对物体进行运动，摄像机会跟随着一起运动保持相对位置不变。

变换Camera位置到原点，上方为Y，观察方向为-Z：

1. 把$\vec{e}$移动到标准位置：$T_{view}=\begin{bmatrix}1& 0& 0& -x_{e}\\ 0& 1& 0& -y_{c}\\ 0& 0& 1& -z_{c}\\ 0& 0& 0& 1\end{bmatrix}$（因为朝原点移动，所以为负）
2. 旋转$\hat{g}$到-Z，$\vec{t}$到Y，$\hat{g}\times\vec{t}$到X：$R_{view}=\begin{bmatrix}x_{\hat{g}\times\hat{t}}& y_{\hat{g}\times\hat{t}}& z_{\hat{g}\times\hat{t}}& 0\\  x_{t}& y_{t}& z_{t}& 0\\x_{-g}& y_{g}& z_{-g}& 0\\ 0& 0& 0& 1\end{bmatrix}$ 

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230704185722.png)

 
模型变换和视图变换经常被一起叫作模型视图变换（ModelView Translation）

## 投影变换（Projection）
投影变换分为两种：
- 正交投影变换：透视线平行
- 透视投影变换：透视线相交，近大远小

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230704193845.png)

## 正交投影



$M_{ortho}=\begin{bmatrix}\frac{2}{r-l}& 0& 0& 0\\ 0& \frac{2}{t-b}& 0& 0\\ 0& 0& \frac{2}{n-f}& 0\\ 0& 0& 0& 1\end{bmatrix}
\begin{bmatrix}1& 0& 0& -\frac{r+L}{2}\\ 0& 1& 0& -\frac{t+b}{2}\\ 0& 0& 1& -\frac{n+f}{2}\\ 0& 0& 0& 1\end{bmatrix}\\\\
=\begin{bmatrix}\frac{2}{r-l}& 0& 0& -\frac{r+l}{r-l}\\ 0& \frac{2}{t-b}& 0& -\frac{t+b}{t-b}\\ 0& 0& \frac{2}{n-f}& -\frac{n+f}{n-f}\\ 0& 0& 0& 1\end{bmatrix}$





正交投影的核心：用一个立方体框住物体的$[l,r]\times[b,t]\times[f,n]$，把这个立方体变换到标准正方体$[-1,1]^{3}$中。

变换顺序：先移动（中点移动到原点），再缩放（基向量缩放比例为$\frac{2}{长/宽/高}$ ）。

透视投影的核心：用“远平面”和“近平面”框住物体，先把“远平面”向“近平面“挤压，然后做一次正交投影。
即透视投影分为两步：
- 将透视投影转化为正交投影
- 将正交投影转换到正则立方体

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230704195505.png)

研究挤压：

规定：
- 挤压过程中，近平面和远平面的z值不发生变换（中间要发生变化）
- 挤压过程中，远平面中心原点$(x,y)^{T}$不发生变化

挤压过程中的x,y变化的比例关系：

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230704195618.png)
x同理。
$y' = \frac{n}{z}y,~~x'=\frac{n}{z}x$

用齐次坐标描述任一点的坐标变换：

$\begin{bmatrix}x\\ y\\ z\\ 1\end{bmatrix}\rightarrow
\begin{bmatrix} nx/z\\ ny/z\\ unknown\\ 1\end{bmatrix}=\begin{bmatrix}nx\\ ny\\ z\cdot unkown\\ z\end{bmatrix}$

把这个变换用齐次坐标矩阵表示：
$M(4\times 4)\begin{bmatrix}x\\ y\\ z\\ 1\end{bmatrix}==\begin{bmatrix}nx\\ ny\\ z\cdot unkown\\ z\end{bmatrix}$

根据矩阵乘法，可以写出M的大致形式：
$M=\begin{bmatrix}n& 0& 0& 0\\ 0& n& 0& 0\\ ?& ?& ?& ?\\ 0& 0& 1& 0\end{bmatrix}$

代入上面提到的两种点：
- 近平面或远平面上的任一点（令$unknown=n,z=n$）：$M\begin{bmatrix}x\\ y\\ n\\ 1\end{bmatrix}=\begin{bmatrix}nx\\ ny\\ n^{2}\\ n\end{bmatrix}$ 根据矩阵乘法行操作：$M第三行\begin{bmatrix}x\\ y\\ n\\ 1\end{bmatrix}=n^{2}$ 因为不涉及旋转，所以第三行与x,y无关。$\begin{bmatrix}0& 0& A& B\end{bmatrix}\begin{bmatrix}x\\ y\\ n \\ 1\end{bmatrix}=n^{2}$ 即：$An+B=n^{2}$ 
- 远平面的原点（令$x=0,y=0,z=f$）：$\begin{bmatrix}0\\ 0\\ f\\ 1\end{bmatrix}
\rightarrow
\begin{bmatrix}0\\ 0\\ f^{2}\\ f\end{bmatrix}$ 同理可得：$Af+B=f^{2}$ 

综上所述，
$A=n+f
B=-nf$

求得变换矩阵为：
$M_{persp\rightarrow -ortho}=\begin{bmatrix}n& 0& 0& 0\\ 0& n& 0& 0\\ 0& 0& n+f& -nf\\ 0& 0& 1& 0\end{bmatrix}$

得到透视投影矩阵为：
$M_{per}=M_{ortho}M_{persp\rightarrow -ortho}\\\\
=\begin{bmatrix}\frac{2}{r-l}& 0& 0& -\frac{r+l}{r-l}\\ 0& \frac{2}{t-b}& 0& -\frac{t+b}{t-b}\\ 0& 0& \frac{2}{n-f}& -\frac{n+f}{n-f}\\ 0& 0& 0& 1\end{bmatrix}\begin{bmatrix}n& 0& 0& 0\\ 0& n& 0& 0\\ 0& 0& n+f& -nf\\ 0& 0& 1& 0\end{bmatrix}\\\\
=\begin{bmatrix}\frac{2n}{r-l}& 0&\frac{l+r}{l-r}& 0\\ 0& \frac{2n}{t-b}& \frac{b+t}{b-t}& 0\\ 0& 0& \frac{f+n}{n-f}& \frac{2fn}{f-n}\\ 0& 0& 1& 0\end{bmatrix}$


# 视口变换
视口变换
将处于标准平面映射到屏幕分辨率范围之内，即[-1,1]^2->[0,width]*[0,height], 其中width和height指屏幕分辨率大小

## 视锥

视锥表示看起来像顶部切割后平行于底部的金字塔的实体形状。这是透视摄像机可以看到和渲染的区域的形状。

定义视锥：
- 长宽比 Aspect
- 垂直的角度 FovY

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230704200231.png)


利用视锥得到物体长宽高：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230704200244.png)


## 屏幕（Screen）
- 二维数组，数组元素为像素
- 典型的光栅成像设备
## 光栅（Raster）
- 德语中的屏幕
- 画在屏幕上
## 像素（Pixel <- PIcture element）
- 像素是一个颜色均匀的小正方形
- 颜色混合而成（红、绿、蓝）
## 屏幕空间
认为屏幕左下角是原点，向右是x，向上是y
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230704200358.png)

规定：
- 像素坐标（Pixel's indices）是(x, y)形式，x, y都是整数。
- 所有的像素都在(0, 0)到(width-1, height-1)之间
- 像素的中心：(x+0.5, y+0.5)
- 整个屏幕覆盖（0，0）to（width，height）


## 视口变换

要做的事情：
先不考虑z轴，把MVP后处于标准立方体$[-1,1]^{3}$映射到屏幕上。即
$[-1, 1]^{2}\rightarrow [0,width]\times [0,height]$

$M_{viewport}=
\begin{bmatrix}
\frac{width}{2}& 0& 0& \frac{width}{2}\\ 
0& \frac{height}{2}& 0& \frac{height}{2}\\
0& 0& 1& 0\\ 
0& 0& 0& 1
\end{bmatrix}$