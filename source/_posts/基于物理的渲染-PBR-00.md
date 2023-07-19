---
title: 基于物理的渲染(PBR)-00
date: 2023-06-13 18:12:12
tags: [图形学]
categories: [PBR]
cover: https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713160112.png
---
**这是一个一直会更新的文章系列**。 学吧，学无止境。

PBR物理光照Shader很多游戏引擎已经自带了， 但是我依旧不知道其中的运行原理，需要好好的梳理下。知其然，并知其所以然。
我们将从物理的角度开始，然后逐步过渡到反射方程和微表面理论，最终推导出Cook-Torrance BRDF模型。
寒霜引擎在SIGGRAPH 2014的分享《Moving Frostbite to PBR》中提出，基于物理的渲染的范畴，由三部分组成：

- 基于物理的材质
- 基于物理的光照
- 基于物理适配的摄像机

【第一章】 开篇：PBR核心知识体系总结与概览

【第二章】 PBR核心理论与渲染光学原理总结

【第三章】 迪士尼原则的BRDF与BSDF相关总结

# 开篇：PBR核心知识体系总结与概览
## 一、PBR核心理论与渲染原理
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713161634.png)
基于物理的渲染（Physically Based Rendering，PBR）是指使用基于物理原理和微平面理论建模的着色/光照模型，以及使用从现实中测量的表面参数来准确表示真实世界材质的渲染理念。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713161717.png)

以下是对PBR基础理念的概括：

- 微平面理论（Microfacet Theory）。微平面理论是将物体表面建模成做无数微观尺度上有随机朝向的理想镜面反射的小平面（microfacet）的理论。在实际的PBR 工作流中，这种物体表面的不规则性用粗糙度贴图或者高光度贴图来表示。

- 能量守恒 （Energy Conservation）。出射光线的能量永远不能超过入射光线的能量。随着粗糙度的上升镜面反射区域的面积会增加，作为平衡，镜面反射区域的平均亮度则会下降。

- 菲涅尔反射（Fresnel Reflectance）。光线以不同角度入射会有不同的反射率。相同的入射角度，不同的物质也会有不同的反射率。万物皆有菲涅尔反射。F0是即0度角入射的菲涅尔反射值。大多数非金属的F0范围是0.02~0.04，大多数金属的F0范围是0.7~1.0。

- 线性空间（Linear Space）。光照计算必须在线性空间完成，shader 中输入的gamma空间的贴图比如漫反射贴图需要被转成线性空间，在具体操作时需要根据不同引擎和渲染器的不同做不同的操作。而描述物体表面属性的贴图如粗糙度，高光贴图，金属贴图等必须保证是线性空间。

- 色调映射（Tone Mapping）。也称色调复制（tone reproduction），是将宽范围的照明级别拟合到屏幕有限色域内的过程。因为基于HDR渲染出来的亮度值会超过显示器能够显示最大亮度，所以需要使用色调映射，将光照结果从HDR转换为显示器能够正常显示的LDR。

物质的光学特性（Substance Optical Properties） 。现实世界中有不同类型的物质可分为三大类：绝缘体（Insulators），半导体（semi-conductors）和导体（conductors）。在渲染和游戏领域，我们一般只对其中的两个感兴趣：导体（金属）和绝缘体（电解质，非金属）。其中非金属具有单色/灰色镜面反射颜色。而金属具有彩色的镜面反射颜色。即非金属的F0是一个float。而金属的F0是一个float3，如下图
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713162408.png)

## 1.1 光与非光学平坦表面的交互原理
在微观尺度上，表面越粗糙，反射越模糊，因为表面取向与整个宏观表面取向的偏离更强。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713163150.png)
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713163201.png)

# 染方程与BxDF
PBR核心知识体系的第二部分是渲染方程与BxDF。渲染方程作为渲染领域中的重要理论，将BxDF代入渲染方程是求解渲染问题的一般方法。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230713164224.png)

写到这写不下去了....... 先鸽， 还是知识太匮乏，希望今年剩下的时间能好好研究下。争取年底前 补完。

