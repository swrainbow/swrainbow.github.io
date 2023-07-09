---
title: HDR
date: 2023-07-08 19:23:41
tags:
---
原文查看拷贝[LearnOpengl章节](https://learnopengl-cn.github.io/05%20Advanced%20Lighting/06%20HDR/)

# HDR why?
一般来说，当存储在帧缓冲(Framebuffer)中时，亮度和颜色的值是默认被限制在0.0到1.0之间的。这个看起来无辜的语句使我们一直将亮度与颜色的值设置在这个范围内，尝试着与场景契合。这样是能够运行的，也能给出还不错的效果。但是如果我们遇上了一个特定的区域，其中有多个亮光源使这些数值总和超过了1.0，又会发生什么呢？答案是这些片段中超过1.0的亮度或者颜色值会被约束在1.0，从而导致场景混成一片，难以分辨：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230709125739.png)

显示器被限制为只能显示值为0.0到1.0间的颜色，但是在光照方程中却没有这个限制。通过使片段的颜色超过1.0，我们有了一个更大的颜色范围，这也被称作HDR(High Dynamic Range, 高动态范围)。有了HDR，亮的东西可以变得非常亮，暗的东西可以变得非常暗，而且充满细节

HDR渲染和其很相似，我们允许用更大范围的颜色值渲染从而获取大范围的黑暗与明亮的场景细节，最后将所有HDR值转换成在[0.0, 1.0]范围的LDR(Low Dynamic Range,低动态范围)。转换HDR值到LDR值得过程叫做色调映射(Tone Mapping)，现在现存有很多的色调映射算法，这些算法致力于在转换过程中保留尽可能多的HDR细节。这些色调映射算法经常会包含一个选择性倾向黑暗或者明亮区域的参数

# ToneMapping
由于现代显示器的动态范围有限，因此无法直接显示 HDR 图像。这就需要进行 tone mapping，即一种将 HDR 图像转换为 LDR 图像的技术。tone mapping 可以调整图像的亮度和对比度，使其适合显示在标准的 LDR 显示器上。常用的 tone mapping 技术包括Reinhard, Neutral, ACES 等。

## Reinhard Tonemapping
Reinhard Tonemapping算法的公式是基于一种对人类视觉感知的假设而设计的。具体来说，这个假设是人类视觉对于亮度的感知是对数函数，而不是线性函数。因此，当我们在LDR显示设备上观察HDR图像时，需要进行一个对数映射的转换，这样才能在LDR显示设备上得到与HDR图像相似的感官体验。Reinhard Tonemapping算法的公式就是一种简单有效的对数映射函数。
Reinhard Tonemapping算法的计算公式如下：

$L_{out}=\frac{L_{in}}{1+L_{in}/L_{white}^{2}}$
其中，$L_{in}$是输入HDR图像的像素值，$L_{out}$是输出LDR图像的像素值，$L_{white}$是输入HDR图像中最亮的像素值。

```c++
float3 ReinhardTonemap(float3 color, float white)
{
    float3 tonemapped_color = color / (1.0f + color / (white * white));
    return tonemapped_color;
}
```

优点：
简单性和良好的表现效果。该算法的运算速度快，能够处理大型HDR图像，同时能够生成高质量的LDR图像。

缺点：
该算法对于局部的亮度变化的适应性较差，可能会导致图像过度饱和或者失真。此外，该算法的输出结果对于不同场景的光照条件的适应性较差，可能需要针对不同场景进行参数调整。

## Neutral Tonemapping
Neutral Tonemapping 算法的公式是基于对每个像素的亮度值进行局部调整的思想而设计的。具体来说，该算法采用平均亮度值$L_{avg}$来调整每个像素的亮度值，使得整张图像的对比度得以保留。
Neutral Tonemmaping算法的公式如下：
$L_{out}\frac{L_{in}}{1+L_{in}/L_{avg}^{2}}$


```c++

float3 NeutralTonemap(float3 color, float avg)
{
    float3 tonemapped_color = color / (1.0f + color / (avg * avg));
    return tonemapped_color;
}

```

优点：
简单性和可扩展性。该算法的运算速度快，能够处理大型HDR图像，同时能够应用于不同类型的场景和图像。

缺点：
该算法可能会导致图像的亮度和色彩失真，特别是在高光和阴影区域。此外，该算法不能处理HDR图像的全局对比度，可能会导致图像过度压缩或者失真。

## ACES
ACES Tonemapping算法是一种基于灰度映射的Tonemapping算法，可以将HDR图像的亮度范围映射到LDR范围内，并保持图像的色彩和对比度。
ACES是一套颜色编码系统，或者说是一个新的颜色空间。它是一个通用的数据交换格式，一方面可以不同的输入设备转成ACES，另一方面可以把ACES在不同的显示设备上正确显示。不管你是LDR，还是HDR，都可以在ACES里表达出来。这就直接解决了VDR的问题，不同设备间都可以互通数据。
然而对于实时渲染来说，没必要用全套ACES。因为第一，没有什么“输入设备”。渲染出来的HDR图像就是个线性的数据，所以直接就在ACES空间中。而输出的时候需要一次tone mapping，转到LDR或另一个HDR。也就是说，我们只要ACES里的非常小的一条路径，而不是纷繁复杂的整套体系。

ACES Tonemapping算法的公式是基于灰度映射的思想而设计的。该算法采用一个控制参数$a$来调整每个像素的亮度值，使得图像的整体对比度得以保留，同时色彩得到修正。
ACES Tonemapping的公式如下：
$L_{out}=\frac{aL_{in}}{1+aL_{in}}$

```c++

float3 ACESTonemap(float3 color, float a)
{
    float3 tonemapped_color = a * color / (1.0f + a * color);
    return tonemapped_color;
}

```

```c++

float3 F(float3 x)
{
	const float A = 0.22f;
	const float B = 0.30f;
	const float C = 0.10f;
	const float D = 0.20f;
	const float E = 0.01f;
	const float F = 0.30f;
 
	return ((x * (A * x + C * B) + D * E) / (x * (A * x + B) + D * F)) - E / F;
}

float3 Uncharted2ToneMapping(float3 color, float adapted_lum)
{
	const float WHITE = 11.2f;
	return F(1.6f * adapted_lum * color) / F(WHITE);
}

```

优点：
ACES Tonemapping算法的优点在于其能够保持图像的色彩和对比度，同时能够映射HDR图像的亮度范围到LDR范围内。此外，该算法适用于各种类型的图像和场景，并且能够适应各种显示设备的亮度和色彩范围。
ACES几乎是最常用的一种秒杀其他tonemapping的算法。

# Color Adjustment
- 色彩调整（Color adjustment）是指对图像中的颜色进行调整，以达到特定的色彩效果或者色彩校正的目的。色彩调整通常包括三个步骤：色彩校正(color correction)、色彩分级(color grading)和色调映射(tone mapping)。
色彩矫正vs色彩分级：
“Correcting is a balance. Grading is a look.”
- 色彩矫正：
色彩校正统一你的电影镜头。
色彩校正使得电影的色调与真实世界的色调保持一致。通过色彩校正，可以让每个视频片段之间的颜色相匹配，从而使它们的色调统一起来。
在色彩矫正过程中，你可以调整诸如曝光、对比度和白平衡，并确保像肤色这样的重要色调得到准确体现。如果你的相机或灯光情况使真实世界中的白色在镜头中出现蓝色，你将在这个阶段纠正这些区域，使其更接近真实的白色。纠正白平衡有助于使你的所有颜色更加真实。
色彩校正不在乎你的风格，而是颜色的正确性。
- 色彩分级：
色彩分级使你的画面更有优势。
在色彩分级阶段，你可以为你影片的着色应用一种整体风格。这为你的项目注入了视觉基调，传达了你希望观众感受到的情感。在调色之前，先进行色彩矫正，以确保开始时有平衡的、自然的色彩。
在色彩矫正过后，可以尝试用色彩分级为影片奠定基调和氛围。如果你的电影是一部残酷的犯罪剧，可以尝试冷色调。而更快乐的视频类型可以尝试用暖色调
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230709132420.png)


