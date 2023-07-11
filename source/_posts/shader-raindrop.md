---
title: shader-raindrop(雨滴)
date: 2023-07-09 18:49:45
tags:
categories: [Shader]
cover: https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230712030750.png
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
```
```glsl
// 绘制雨滴圆形
float d = length(uv);
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230712022056.png)


```glsl
// 分割 并且获取每个格子ID
UV *= 0.5;
vec2 id = floor(UV);
```

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230712022223.png)

```glsl
// 打散每个格子
vec3 n = N13(id.x * 107.45 + id.y * 3543.654);
vec2 p = (n.xy - .5) * .7;
float d = length(uv - p)
```

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230712022552.png)


```glsl
#define S(a, b, t) smoothstep(a, b, t)

float Saw(float b, float t) {
  return S(0., b, t)*S(1., b, t);
}
// 随记雨滴渐入渐出时间
float fade = Saw(0.025， fract(t + n.z));

// 降低雨滴出现的概率
float c= S(.3, 0., d) * fract(n.z * 10)*fade;
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230712023412.png)

## 动态雨滴

```glsl
vec2 a = vec2(5., 1.); // 
vec2 grid = a * 2.;
vec2 id = floor(uv * grid);
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230712024014.png)


```glsl
float colShift = N(id.x); // N是一个伪随机函数
uv.y += colShift; // 沿着y轴偏移
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230712024844.png)

```glsl
// y轴上下运动
float y = UV.y * 20.;
y = (Saw(.85, t)- .5) *.9 + .5;
float d = length((st-p)*a.yx);

//整体画布跟着运动
uv.y += t*0.75 
```

```glsl
// y轴上下运动
float x = n.x - .5;
float ti = fract(t + n.z);

y = (Saw(.85, ti) - .5) * .9 + .5;
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230712025315.png)



```glsl
// 雨滴落痕
float r = sqrt(S(1., y, st.y));
float cd = abs(st.x - x);
float trail = S(.23, .15, cd);

// 截去雨滴前面的痕迹
float trailFront = S(-.02, .02, st.y - y);
trail *= trailFront * r * r;
```

```glsl
// 雨滴形状塑造
float trail = S(.23 * r, .15 *r*r, cd);
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230712025725.png)

```glsl
// 增加雨滴路径自然扭曲
float wiggle = sin(y + sin(y));
x += wiggle * (.5 - abs(x)) * (n.z - .5);
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230712025855.png)

```glsl
// 增加雨滴路径上的小水滴
y = fract(y * 10.) + (st.y - .5);
float dd = length(st - vec2(x,y));
float droplets = S(.3, 0., dd);
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230712030023.png)

## 法线
```glsl
//内置求到函数
vec2 n = vec2(dfdx(c.x), dfdy(c.x));
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/%E6%88%AA%E5%B1%8F2023-07-12%2003.03.00.png)

## 后处理

```glsl
// 把色调改成暗蓝色
col *= vec3（。8，。9，1.3）

// 暗角
UV -= .5;
col *= 1. - dot(UV, UV);
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230712030542.png)











