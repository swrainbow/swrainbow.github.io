---
title: Games101-着色（插值、高级纹理映射）
date: 2023-07-05 17:37:48
tags: [图形学， Shading]
categories: [games101 系列]
---

# 重心坐标
## 基础定义
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705173817.png)

三角形ABC平面内任意一点(x,y)都可以写成这三点坐标的线性组合形式，即$(x,y)=\alpha A+\beta B+\gamma C$，当$\alpha+\beta+\gamma=1$时，称点(x,y)为重心坐标。
其中，当(x,y)为三角形内一点时，$\alpha,\beta,\gamma>=0$

根据这个定义，可以求得顶点的重心坐标。
如A：$\alpha=1,\beta=0,\gamma=0$，BC同理。

## 几何定义
几何观点——利用面积比求重心坐标

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705173853.png)

由这个定义，可以求出重心的坐标。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705173910.png)

## 重心坐标公式
可以通过几何定义和行列式求出重心坐标的公式：

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705173929.png)


# 插值
插值的作用：知道三角形顶点属性，三角形内部进行平滑过渡。
插值的属性：纹理坐标，颜色，法向量等等...

插值的方法：利用重心坐标进行插值。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705174017.png)

用要插值的属性代替$V_{i}$

- 重心坐标不能保证投影后不变，所以三维的要在三维中找到重心坐标再做插值，而不能投影后做插值

我们的重心坐标往往都是在屏幕空间下所得到的，如果直接使用屏幕空间下的重心坐标进行插值会造成一定的误差，与在view space下是不一样的。
具体解决方案：
https://zhuanlan.zhihu.com/p/144331875

这一点在作业3中有所体现。

# 纹理过大过小的问题及解决方案
## 纹理过小
纹理过小会导致失真，因为屏幕空间几个pixel对应在纹理贴图UV坐标上都是一个texture，往往会导致严重的走样。
我们知道，可以将屏幕或者画布上的一个单元称之为像素 pixel，为了与纹理上的一个单元进行区别，特将后者称之为 纹理像素texel。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705174148.png)

如图，红点是屏幕空间下一pixel对应的texel，一种想法是去取离它最近的点，这种想法是不可取的，会出现走样现象。
## 双线性插值
一种缓解因纹理过小带来的走样现象的方法是双线性插值。如图：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705174213.png)

- 取texel最近的四个点
- 做上下双线性插值（水平两次，左右一次）或左右双线性插值

### 双向三次插值
相邻的16个点做水平和竖直的差值，利用三次方程进行两次插值，效果更好，但是计算速度很低。
### 方法对比
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705174253.png)

## 纹理过大
纹理过大时，一个pixel对应多个texel，导致采样频率不足，最终出现摩尔纹和锯齿的走样现象。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705174313.png)

远处地平面的一个像素，对应一个大块的纹理，简单的点采样不行。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705174406.png)

footprint：称一个pixel覆盖不同数量的texel的现象为footprint。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705174431.png)

如上图所示一个屏幕空间的蓝色像素点离相机越远，对应在texture空间的范围也就越大，覆盖越来越多的texel。其实也就是越来越欠采样。

例：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705174449.png)

近处圆圈中的footprint就比远处圆圈中的footprint小。

### Mipmap

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705174517.png)

- Mipmap的核心想法：离线预处理查询每个footprint对应区域里的颜色均值，在渲染前生成。
- Level：因为一个场景中存在不同大小的footprint，需要为不同大小的footprint准备不同精度的mipmap，称为Level。因此越高的level就代表了更大的footprint的区域查询。 
  - Level的层数：$log_{2}n$
  - 存储量：$\large \frac{4}{3}$（1+1/4+1/16...） 
    - 一种理解方式：新增的全部被压缩到1/3处。
    ![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705174601.png)

- 快；不准；
- 只能做近似正方形的查询

计算使用Level等级D：利用屏幕像素的相邻像素点估算footprint大小再确定level等级。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705174729.png)
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705174740.png)

在屏幕空间中取当前像素点的右方和上方的两个相邻像素点(4个全取也可以)，分别查询得到这3个点对应在Texture space的坐标，计算出当前像素点与右方像素点和上方像素点在Texture space的距离，二者取最大值，计算公式如图中所示，那么levelD就是这个距离的log2值 ($D=log_{2}L$) 。
这里将L近似成正方形的边长。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705174756.png)
这样计算出来的D是一个连续值而非整数，有两种解决方法：
- 四舍五入取得最近的那个level D
- 利用D值在 向下和向上取整的两个不同level进行3线性插值

四舍五入在颜色过渡上会产生突兀

重点是三线性插值。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705174818.png)

三线性插值即对DLevel进行一次双线性插值，再对D+1Level进行一次双线性插值，然后对这两个结果进行一次线性插值，最终结果为双线性插值。
开销：两次查询一次插值。

然而Mipmap也有局限性，正如前文提到Mipmap只能预处理出正方形区域的均值，这导致远处过度模糊。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705174840.png)

### 各向异性滤波Mipmap
真实情况屏幕空间上的区域对应在纹理区域上即footprint并不一定是正方形。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705174902.png)

解决方法：各向异性过滤
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705175002.png)

- 允许对长条的区域进行范围查询，但是不能用于斜着的区域
- 生成各向异性过滤的图（Ripmaps）的开销是原本的三倍
- 各向异性的意思是，在不同的方向上它的表现各不相同
- 各向异性的几X是压缩几倍，也就是从左上角往右下角多几层

得到的结果比Mipmap更好。

### EWA过滤
把任意不规则的形状拆成很多不同的圆形，去覆盖这个形状，多次查询自然可以覆盖，但是耗时大

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705175042.png)