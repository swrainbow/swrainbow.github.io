---
title: Early-Z和Z-Prepass
date: 2023-05-11 19:54:24
tags: [图形学]
categories: [图形学]
---
后续需要改进， 对相关概念理解还是不清楚（待改进文章）
# 深度测试
深度测试在管线中位于像素阶段的合并子阶段。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711195522.png)
具体来说片元着色器后，模板测试前。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711195632.png)

深度测试用于解决片元的可见性问题。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711195706.png)
如上图，由于深度值越小，代表离相机越近。假设首先渲染深度全为5的三角形。此时深度缓冲区初始深度为inf，所有片元通过并将深度缓冲区更新为5。接下来渲染深度进行插值的三角形，当插值深度小于深度缓冲区时，通过测试并更新。否则，抛弃该片元。

深度测试的伪代码如下：
```c++
for(each trianle T){
	for(each fragment(x, y, z) in T){
		if(fragment.z < ZBuffer[x, y]){
			FrameBuffer[x, y] = fragment.rgb;
			ZBuffer[x, y] = fragment.z;
		}
		else{
			abort;
		}
	}
}
```
由于深度测试的位置在片元着色器后，输出阶段前，这会导致会被深度测试抛弃的片元也经过了前面的管线流程，进行光照计算。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711195813.png)
下面讨论两种解决方法，Early-Z和Z-Prepass。
# Early-Z
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711195919.png)
在光栅化阶段，三角形被光栅化为片元。在提前深度测试阶段中，每个片元执行深度测试以确定其是否可见。如果不可见，则抛弃该片元，不进行后续的片元着色器和光照计算。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711200042.png)
对于不透明物体，如果从远往后渲染，则Early-Z总是通过，前面物体的片元覆盖后面物体的片元，造成overdraw。所以，为了更高效的发挥Early-Z，不透明物体的正确渲染顺序应该是 从前往后。
一种排序流程是让CPU先将物体由近到远排序，然后再发送给GPU渲染。然而，大量的物体深度排序可能造成CPU消耗过高的问题
除此之外，一些操作也会导致Early-Z失效。
- 开启AlphaTest或clip/discard等手动丢弃片元操作 
  - 通常Early-Z不仅会进行深度测试，还要进行深度写入。如果通过Early-Z测试并写入深度的片元在片元着色器中被丢弃，那么后续比它深度大的片元不会通过测试，不会正常渲染。
- 手动修改GPU插值的深度 
  - 类似上述情况。
- 开启Alpha Blend 
  - 一般情况下，Alpha Blend不会开启深度写入。
- 关闭深度测试

# Z-Prepass
Z-prepass的目的是在执行主要渲染通道之前，首先填充深度缓冲区。它通过渲染场景的物体，计算每个像素的深度值，并将其存储在深度缓冲区中。后续pass可以使用这些预先计算的深度值来决定哪些像素应该被渲染，以避免不必要的像素片段的绘制和深度测试。

## 双Pass法
- Pass0: Z-Prepass 
  - 仅仅写入深度，不输出任何颜色。该通道的目的是将物体的深度值写入深度缓冲区。
- Pass1: 正常渲染 
  ○ 关闭深度写入操作，并将深度写入函数设置为相等。在这个通道中，可以进行正常的AlphaBlend操作，同时考虑深度值。

通过使用双Pass法，每个物体都会经过两个渲染通道的处理。其中，Z-prepass阶段的结果将自动形成一个包含最小深度值的Z-buffer缓冲区，无需CPU进行显式的深度排序操作。这样可以大大减少对CPU的负担，并提高渲染性能。

使用双pass法，会带来两个问题
- 动态批处理 
  - 多pass shader无法进行动态批处理。
- Draw Call 
  - 使用Z-Prepass的物体，会额外产生一倍的Draw Call。
