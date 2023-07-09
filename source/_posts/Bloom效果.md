---
title: Bloom效果
date: 2023-07-06 20:22:40
tags:
---
# 什么是bloom 效果
Bloom 也称为辉光效果
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230708175631.png)
模拟摄像机的一种图像效果， 让物体具有真实的明亮效果
Bloom和HDR结合使用效果很好。常见的一个误解是HDR和泛光是一样的，很多人认为两种技术是可以互换的。但是它们是两种不同的技术，用于各自不同的目的上。可以使用默认的8位精确度的帧缓冲，也可以在不使用泛光效果的时候，使用HDR。只不过在有了HDR之后再实现泛光就更简单了。

- LDR
    - JPG PNG 格式图片
    - RGB 范围在0-1
- HDR
    - HDR, EXR 格式图片
    - RGB 范围可超过1

- 高斯模糊
    - 减少图像噪声 降低细节层次

- 卷积
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230708175855.png)


# 实现思路
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230708175843.png)