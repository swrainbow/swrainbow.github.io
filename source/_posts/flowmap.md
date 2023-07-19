---
title: flowmap
date: 2023-06-25 01:30:44
tags: [图形学]
categories: [图形学]
cover: https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230718210433.png
---

# Flowmap
我们知道，模型动画可以通过对纹理采样的uv偏移达到，这里的纹理可以是反照率纹理，也可以是法线纹理。

```glsl
uv -= uvOffset;
sample(texture, uv);
```

我们可以将uvOffset看作一个方向向量dir，flowmap的基本思想就是存储每个texel的dir信息，达到不同片元不同uv偏移的效果。注意，这个dir不仅存储方向，也存储强度。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230710184205.png)
如图所示，我们首先确定一个向量场，再将这个向量场用对应的颜色绘制出来。通常，rg通道分别表示u, v方向的偏移，以128为中心。

# 扰动flow
扰动flow就是利用这些dir对原UV轴直接进行扰动，从而实现对纹理的扭曲效果。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230710184519.png)
扰动flow要求采样的纹理不论如何变形，它都必须看起来很好。这只有在各向同性的模式下才能实现。各向同性是指图像在所有方向上看起来都差不多。扭曲效应对非常湍急或非常迟缓的水流效果最好。它对表现出清晰波纹模式的更平静的流动效果不好，因为这种波纹有明确的方向。它们是各向异性的。

## 时间性走样
为了进行UV动画，我们需要将扰动强度与时间相关联。即

```glsl
uv -= flowDir * time;
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/flowmap2.gif)
然而，随着时间的增长，扰动强度会越变越大，导致邻域像素在纹理上的texel位置偏差越来越大，从而造成严重的走样。由于这种走样由于时间引起，这里称它为“时间性走样"。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230710184719.png)
为了解决时间性走样，我们需要限制偏移的强度，一种想法就是将time钳位为0~1的周期函数，即

```glsl
phase = frac(time);
uv -= flowDir * phase;
```

phase的图像如下，它是一个周期为1的锯齿函数。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230710184814.png)

## 时间性突变
我们解决了时间性走样，但这带来了时间性突变的问题。这是由于采样强度始终相同，导致周期结束与周期开始时的变化显而易见。所以我们需要在周期结束和开始时都淡化纹理，让他们从黑色渐变到最明显，再渐变回黑色。
也就是说，我们需要构造一个函数$w(p)$，使得$w(1*k)=0, w(1*k+1/2)=1$。这个$w(p)$可以是三角形函数$w(p)=1-|1-2p|$。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230710184925.png)
然而，这样做的突变还是很明显。这是因为黑色缓冲应用到了所有片元。我们应该将黑色缓冲随着时间分散到各个片元，从而减少突变。我们将noise放到flowmap的a通道，并用它对时间进行偏移，从而对$w(p)$的值进行扰动。并且，时间上的偏移也使变形的进展不均匀，导致整体上的变形更加多样。

```glsl
phase = frac(time+noise);
uv -= flowDir * phase;
```
然而，图案上有黑色始终是我们不想出现的结果，更好的方式是将这些黑色替换为纹理上的图案。而替换的纹理不能和原纹理有重叠部分，否则就会出现原纹理的淡入淡出。
也就是说tex = texA*wA + texB*wB。我们希望的结果是当纹理A采样强度wA最弱时，纹理B采样强度wB最强，反之亦然。这可以通过将wA的图像偏移半周期做到。如下图，这是将采样偏移半周期后图像。红色三角波代表wA，红色曲线代表texA采样函数。蓝色三角波代表wB，蓝色曲线代表texB采样函数。注意，这里也将texB的采样函数偏移半周期，让texA和texB没有重叠部分。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230710185033.png)
```glsl
phaseA = frac(time);
phaesB = frac(time+0.5);
texA = sample(texture, uv - flowDir * phase0);
texB = sample(texture, uv - flowDir * phase1);
wA = 1 - abs(1-2*phaseA);
wB = 1 - abs(1-2*phaseB);
tex = texA*wA + texB*wB;
```

这样还有一个小缺陷，就是这样做将周期从1变为了0.5。观察上图，tex的过程如下，前半段：前半tex淡入，后半tex淡出；后半段：后半tex淡出，前半tex淡入。前后两段实际上是相同的。
解决方案很简单，让texB采样的UV本来就与texA不同。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230710185127.png)
此时的过程如下，前半段：texA前半淡入，texB后半淡出；后半段：texA后半淡出，texB前半淡入。这就形成了一个完整的周期。
```glsl
phaseA = frac(time);
phaesB = frac(time+0.5);
texA = sample(texture, uv - flowDir * phase0);
texB = sample(texture, uv - flowDir * phase1 + 0.5);
wA = 1 - abs(1-2*phaseA);
wB = 1 - abs(1-2*phaseB);
tex = texA*wA + texB*wB;
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/flowmap.gif)