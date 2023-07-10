---
title: shader-circle
date: 2022-07-10 01:17:46
tags:
categories: [Shader]
---

画圆的方法有很多，这边文章使用了一种比较通用的方法，这也是我在2d gameEngine中使用的。废话不多说，马上开始。

# 开始
我这里使用shadertoy 编写shader 主要是方便。
准备工作， 保持宽高比，将中心从左下角转移到屏幕中心
```c++
vec2 uv = fragCoord/iResolution.xy * 2.0 - 1.0;
float aspect = iResulution.x / iResulution.y
uv.x *= aspect
```

## 画个简单的圆
```glsl
float distance = length(uv);
fragColor.rgb = vec3(distance);
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230710123232.png)

我们可以翻转下
float distance = 1 - length(uv);
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230710123546.png)

可以看到边缘有模糊过度，我们不想要。可以使用step或者其他判断条件
```glsl
col = vec3（step(0.0, distance)）
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230710123850.png)

现在有一个简单的圆形了

进一步， 我想要实现类似甜甜圈这样的造型：
```glsl
col = vec3（step(0.0, distance)）
// 注意前面已经把距离反转了
if(distance > 0.02) {
    col = vec3(0.0);
}
```
可以把上面的if判断改成step
```glsl
col = vec3（step(0.0, distance)）
// 注意前面已经把距离反转了
col *= vec3(1.0 - step(0.02, distance));
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230710124926.png)

更通用点， 我们可以把0.02作为一个参数提取
float thickness = iMouse.x / iResulution.x;
```glsl
col = vec3（step(0.0, distance)）
// 注意前面已经把距离反转了
col *= vec3(1.0 - step(thickness, distance));
```

现在我想要圆的边框能平滑过度，这也可以防止放大后的锯齿现象
我们可以使用smoothstep代替step

```glsl
float thickness = iMouse.x / iResulution.x;
float fade = 0.002;
col = vec3（smoothstep(0.0, fade,  distance)）；
// 注意前面已经把距离反转了
col *= vec3(1.0 - smoothstep(thickness, thickness - fade, distance));
```

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230710125643.png)

最后，一个nice smooth circle
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230710125849.png)