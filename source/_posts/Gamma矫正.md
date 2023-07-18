---
title: Gamma矫正
date: 2023-07-06 20:04:34
tags: [图形学]
categories: [图形学]
cover: https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713154138.png
---
原视频： [韩世麟：Gamma校正与线性工作流入门讲解](https://www.bilibili.com/video/BV15t411Y7cf/?vd_source=dc8bc359da1cacb67fa6054d14862294)
# 使用伽马矫正的原因
## 原因1：人眼对光强的非线性感知
如图，人眼在接受物理光照时，感知到的灰度并不是与能量强度线性映射的，而是对其进行了一个"上拱"幂函数的转换，导致人眼对低能量的物理光感知变换更加敏感，采样频率更低。


![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230708211203.png)
人眼感知：上面灰度渐变均匀，基于美术感知，受到环境等因素的影响。物理上灰度非线性增长。人眼感知的中灰度又称"美术中灰“。
物理测量：下面灰度渐变均匀，基于能量测量。物理上灰度线性增长。由于人眼对低物理能量采样频率更低，所以感知到的物理暗部更少，物理亮部更多。
在下面物理感知中，物理中灰度对应人眼感知中灰度偏亮色度。理论上，人眼感知中灰度为物理白色度的21.8%。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230708211255.png)

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230708211404.png)
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230708211423.png)

作用：将人眼感知灰度与物理测量灰度相互转换。如上图，如果将物理物理测量灰度值$x$作为横轴，人眼感知灰度值$y$作为纵轴，则$y-x$曲线即为伽马曲线，表示人眼感知灰度与物理测量灰度的函数关系，$y=x^{gamma}$。

## 原因2：数字图像带宽
数字图像所能采集和回放的灰阶层次是有限的，需要节约带宽。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230708211457.png)


如果均匀采样美术灰阶，人眼中的亮部暗部一样多，各占128灰阶。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230708211538.png)

如果均匀采样物理灰阶，人眼中的亮部采样频率大于暗部采样频率，美术暗部所占灰阶少于美术亮部。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230708211610.png)
具体来说，如果均匀采样美术灰阶，美术暗部和亮部各占128阶。如果均匀采样物理灰阶，美术暗部只占56阶，而美术亮部占200阶。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230708211626.png)

如果计算机存储图像文件时，采用线性的物理灰阶，由于美术暗部采样频率少，所以人眼感知的暗部变化更加突兀，出现断层的现象。为了解决这个问题，我们需要给予美术暗部更多的采样频率，即从物理空间转换到人眼感知空间，即利用伽马矫正进行转换。

# 伽马函数

伽马函数：幂函数，$color_{gamma}=color_{linear}^{gamma}$。
利用幂函数性质，如果进行两次互为倒数的幂函数，则还原为参数。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230708211744.png)

# 贴图流程
错误贴图采样流程：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230708211918.png)

直接线性采样贴图，得到的是伽马矫正后的偏亮的纹理。物理计算后进行伽马矫正，纹理回归正常，光照偏暗。进行伽马矫正， 光照正常，纹理偏亮。

正确贴图采样流程：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230708212014.png)
线性工作流，将贴图Degamma再进行采样。