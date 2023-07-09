---
title: Games101-几何-贝塞尔曲线
date: 2023-07-06 18:06:45
tags:
---

# 贝塞尔曲线
## 定义
贝塞尔曲线：由控制点和线段组成的曲线，控制点是可拖动的支点。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706180720.png)
如图，蓝色为贝塞尔曲线，$p_{1},p_{2},p_{3}$为控制点，曲线和初始与终止端点相切，并且经过起点$p_{0}$与终点$p_{3}$。

## de Casteljau Algorithm
de Casteljau算法描述了如何用多个点画出一条贝塞尔曲线。
其核心是线性插值和递归。
贝塞尔曲线的定义很像参数方程，给定一个参数$t\in[0,1]$就能确定贝塞尔曲线上的一点，倘若取完所有t值，就能得到完整的贝塞尔曲线。
一阶贝塞尔曲线
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706180951.png)
通过线性插值，可以算出线段上点的坐标：

$
B_{1}(t)=P_{0}+(P_{1}-P_{0})t\\ 
B_{1}(t) = (1-t)P_{0}+tP_{1}, t\in
$
二阶贝塞尔曲线
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706181049.png)
首先在$b_{0}b_{1}$上进行一阶贝塞尔曲线得到点$b_{0}^{1}$，再在$b_{1}b_{2}$上进行一阶贝塞尔曲线得到点$b_{1}^{1}$，再在$b_{0}^{1}b_{1}^{1}$上进行一阶贝塞尔曲线得到点$b_{0}$。之后再连接$b_{0}b_{0}^{2}b_{2}$就可以得到蓝色的贝塞尔曲线。

三阶贝塞尔曲线
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706181109.png)
## 代数式表达
将贝塞尔曲线展开可以得到n阶贝塞尔曲线的代数表达式：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706181137.png)
注：

$\large
\left(\begin{matrix}n\\ i\end{matrix}\right)=C_{n}^{i}$

以二阶贝塞尔曲线展开为例
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706181256.png)


## 性质
- 必定经过起始与终止控制点
- 必定经与起始与终止线段相切
- 具有仿射变换性质，可以通过移动控制点移动整条曲线
- 凸包性质，曲线一定不会超出所有控制点构成的多边形范围 
  - 凸包：墙上许多钉子，用一条橡皮筋包住最外边的钉子，再松手，橡皮筋收缩后的外框就是凸包。
  ![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706181451.png)

## 分段贝塞尔曲线
传统贝塞尔曲线的缺点：当控制点多的时候不好控制曲线的形状。

分段贝塞尔曲线：将一条高次曲线分成多条低次曲线的拼接，其中用的最多的便是用很多的3次曲线来拼接。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706181527.png)
分段贝塞尔曲线的连续性：$C_{n}$连续，表示n阶导数连续。

# 贝塞尔曲面
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706181557.png)
以$4\times 4$控制点的贝塞尔曲面为例：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706181620.png)
1. 在这4个控制点之下利用第一个参数 u 运用第一章的计算贝塞尔曲线的方法得到蓝色点，因为有4列，所以一共可以得到如图所示的4个蓝色点。(灰色曲线分别为每列4个点所对应的贝塞尔曲线)
2. 在得到4个蓝色顶点之后，在这四个蓝色顶点的基础之下利用第二个参数 v 便可以成功得出贝塞尔曲面上一个点
3. 遍历所有的 u，v值就可以成功得到一个贝塞尔曲面

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230706181644.png)

