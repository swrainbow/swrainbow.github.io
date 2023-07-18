---
title: 光线追踪（基本原理)
date: 2023-07-05 19:57:01
tags: [图形学]
categories: [games101 系列]
cover: https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230718210215.png
---
# 引入光线追踪的原因
光栅化的缺点：不能很好的处理全局光照
- 光栅化：快 real-time 质量低
- 光线追踪：慢 offline 质量高

下面考虑全局光照的例子：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705195757.png)

# Whitted-Style 光线追踪
## 定义光线
光线的定义：
- 直线传播
- 光线间不会碰撞
- 光线从光源发出，进入场景不断碰撞，最终到达眼睛

光线传播性质reversal-reciprocity可逆性：
光从光源发出进行弹射，最终进入眼睛。也可以认为眼睛发出一些感知光线进行弹射，最终回到光源（逆过程）。
## Ray Casting 光线投射
光线投射用来判断是否需要进行光线追踪还是直接进行着色模型。
将成像平面化为不同pixel，对于每个pixel，从眼睛开始穿过这个pixel发出一条光线，这条光线最终打到某个位置，与物体相交或不相交。
如果和某个物体相交，说明眼睛朝这个方向能看到这里。再让物体上交点与光线连接，如果没有遮挡，就形成了一条有效的光路，否则为阴影。
- 光源$\rightarrow$物体某点$\rightarrow$眼睛
- 比shadow mapping方便
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705195914.png)

紧接着，用着色模型对这个点进行局部光照模型计算，得到该像素的颜色，那么遍历所有近投影平面上的像素就能得到一张完整的图像。但如果光线追踪仅仅是在第一步Ray Casting就停止的话，那么它的效果与局部光照模型是一样的，因此我们需要第二步，真正的考虑全局效果。
## Whitted Style Ray Tracing
两个假设前提：
- 人眼是一个点
- 场景中的物体，光线打到后都会进行完美的反射/折射

定义三种光线：
- primary ray：眼睛直接发出的光线
- secondary ray：间接发出的光线
- shadow ray：物体交点间接发出到光源的光线
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705200052.png)

因此每一个交点的颜色贡献来自这样种几类型 直接光照，反射方向间接光，折射方向间接光（如果有折射的话）
- 递归过程 
  - 递归出口例如允许的最大反射或折射次数为10
- 能量损耗 
  - 系数决定，因此越往后的折射和反射光贡献的能量越小，这也是为什么在上文中提到根据光线能量权重求和。 e.g. 反射系数为0.7，那么第一次反射折损30%，第二次反射折损1-（70%x70%），依次类推。
- 如果反射或折射光线没有碰撞到物体，一般直接返回一个背景色

# 光线的表示
将光线类比成一条射线，射线上的点由起点和方向向量决定：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705200153.png)

# 光线与物体的交点
## 隐式几何

例：求光线与球地交点
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705200322.png)
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705200249.png)

## 显式几何
真正在图形学中大量运用的其实是显示曲面，更具体来说就是许许多多个三角形，因此如何判断一条光线与显示曲面的交点，其实也就是计算光线与三角形面的交点。

一种思路：光线与平面内每个三角形判断交点 慢 开销大
把问题分成：光线是否和平面有交点 交点是否在三角形内部
定义平面：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705200406.png)
联立方程：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705200424.png)
一步到位：MT算法
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705200441.png)

- 用重心坐标描述三角形
- 线性方程组$\rightarrow$克莱姆法则

# 反射与折射
这部分内容来自知乎[孙小磊](https://zhuanlan.zhihu.com/p/144403005)
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705200605.png)