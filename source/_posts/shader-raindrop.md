---
title: shader-raindrop(雨滴)
date: 2023-07-09 18:49:45
tags:
---
# 雨滴效果
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230710011037.png)
- 第一层： 静态雨滴
- 第二层： 动态雨滴
- 第三层： 动态雨滴（小）
- 雨滴法线
- 后处理（颜色滤镜）
- 后处理（暗角滤镜）

## 静态雨滴
```glsl
// 坐标范围 -0.5 ～ 0.5 原点在画布中心
vec2 uv = (fragCoord.xy-.5*iResolution.xy) / iResolution.y;

// 坐标范围 0 ～ 1 原点在画布左下角
vec2 UV = fragCoord.xy/iResolution.xy;

// 绘制雨滴圆形
float d = length(uv);

vec3 n = N13(id.x * 107.45 + id.y * 3543.654);
vec2 p = (n.xy - .5) * .7;
float d = length(uv - p)


```glsl
// 随记雨滴渐入渐出时间
float fade = Saw(0.025， fract(t + n.z));

// 降低雨滴出现的概率
float c= S(.3, 0., d) * fract(n.z * 10)*fade;
```






