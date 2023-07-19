---
title: raytracing-in-one-week
date: 2023-07-13 01:31:26
tags: [光线追踪]
categories: [光线追踪]
cover: https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711174032.png
---
# 开始
文章参考: [Ray Tracing in One Weekend](https://raytracing.github.io/books/RayTracingInOneWeekend.html)

一些翻译和整理，建议大家看原文。

## 图片输出

需要一种查看图像的方式。最直接的方法是将其写入文件中。问题在于，有很多格式可供选择。其中许多是复杂的。我通常会从一个纯文本的PPM文件开始。以下是来自维基百科的一个简洁描述：
PPM图像格式（Portable Pixmap）是一种用于存储二维灰度和彩色图像的文件格式。它是一种无损的、像素级的图像格式，以纯文本形式保存。PPM格式是一种简单而广泛支持的图像格式，它不依赖于任何特定的图像编辑软件。该格式的文件以".ppm"作为扩展名。PPM文件的结构简单明了，它以ASCII字符表示像素值，可以轻松地在文本编辑器中编辑和读取。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711161936.png)

```c++
#include <iostream>

int main() {

    // Image

    const int image_width = 256;
    const int image_height = 256;

    // Render

    std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";

    for (int j = image_height-1; j >= 0; --j) {
        for (int i = 0; i < image_width; ++i) {
            auto r = double(i) / (image_width-1);
            auto g = double(j) / (image_height-1);
            auto b = 0.25;

            int ir = static_cast<int>(255.999 * r);
            int ig = static_cast<int>(255.999 * g);
            int ib = static_cast<int>(255.999 * b);

            std::cout << ir << ' ' << ig << ' ' << ib << '\n';
        }
    }
}
```

- 像素按照从左到右的顺序逐行写出。
- 行按照从上到下的顺序写出。
- 按照惯例，每个红/绿/蓝分量的取值范围为0.0到1.0。我们将在内部使用高动态范围时放宽这一限制，但在输出之前，我们将进行色调映射，将其调整到0到1的范围内，因此此代码不会改变。
- 红色从完全关闭（黑色）到完全打开（鲜红色）从左到右变化，绿色从底部的黑色变化到顶部的完全打开。红色和绿色结合在一起会产生黄色，所以我们应该期望右上角是黄色的。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711162201.png)

打开文本编辑器查看，上面的图片就是下面的数据段
```txt
P3
256 256
255
0 255 63
1 255 63
2 255 63
3 255 63
4 255 63
5 255 63
6 255 63
7 255 63
8 255 63
9 255 63
...
```

## Rays, a Simple Camera, and Background

前提你需要构建一些vec3、 random 之类的结构， 原文中有介绍。这里就不再赘述了。

### Rays
如何构造光线（点光源）， 就是光线就是起点和光线方向还有时间的函数 也就是ray = ray.origin + ray.dir * time

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711163032.png)

```c++
#ifndef RAY_H
#define RAY_H

#include "vec3.h"

class ray {
    public:
        ray() {}
        ray(const point3& origin, const vec3& direction)
            : orig(origin), dir(direction)
        {}

        point3 origin() const  { return orig; }
        vec3 direction() const { return dir; }

        point3 at(double t) const {
            return orig + t*dir;
        }

    public:
        point3 orig;
        vec3 dir;
};

#endif
```

### sending ray into the scene 
现在准备进入光线追踪的领域了。在核心部分，光线追踪器通过像素发射光线，并计算沿着这些光线方向所见的颜色。其中涉及的步骤有：
（1）计算从眼睛到像素的光线，（2）确定光线与哪些物体相交，以及（3）计算相交点的颜色。在初步开发光线追踪器时，我总是使用一个简单的相机来让代码快速运行起来。我还会创建一个简单的ray_color(ray)函数，用于返回背景的颜色（一个简单的渐变色）。

在调试时，我经常因为频繁转置 𝑥 和 𝑦 而遇到问题，所以我将使用一个非方形的图像。目前，我们将使用16:9的宽高比，因为这是非常常见的。

除了设置渲染图像的像素尺寸之外，我们还需要设置一个虚拟视口，通过该视口传递我们的场景光线。对于标准的方形像素间距，视口的宽高比应与我们渲染的图像相同。我们选择一个高度为两个单位的视口。我们还将设置投影平面与投影点之间的距离为一个单位。这被称为“焦距”，不要与后面将介绍的“对焦距离”混淆。

我将把“眼睛”（或者如果你想象成相机的话，就是相机中心）放在(0,0,0)处。我让y轴朝上，x轴朝右。为了遵守右手坐标系的约定，向屏幕内部是负的z轴。我将从左上角开始遍历屏幕，并使用两个屏幕边缘的偏移向量来移动光线的终点位置。请注意，我没有将光线方向设置为单位长度向量，因为我认为不这样做可以使代码更简单且稍微更快。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711163416.png)

```c++
color ray_color(const ray& r) {
    vec3 unit_direction = unit_vector(r.direction());
    auto t = 0.5*(unit_direction.y() + 1.0);
    // 当然可以使用lerp
    return (1.0-t)*color(1.0, 1.0, 1.0) + t*color(0.5, 0.7, 1.0);
}

int main() {

    // Image
    const auto aspect_ratio = 16.0 / 9.0;
    const int image_width = 400;
    const int image_height = static_cast<int>(image_width / aspect_ratio);

    // Camera

    auto viewport_height = 2.0;
    auto viewport_width = aspect_ratio * viewport_height;
    auto focal_length = 1.0;

    auto origin = point3(0, 0, 0);
    auto horizontal = vec3(viewport_width, 0, 0);
    auto vertical = vec3(0, viewport_height, 0);
    //辅助生成光线方向
    auto lower_left_corner = origin - horizontal/2 - vertical/2 - vec3(0, 0, focal_length);

    // Render

    std::cout << "P3\n" << image_width << " " << image_height << "\n255\n";

    for (int j = image_height-1; j >= 0; --j) {
        std::cerr << "\rScanlines remaining: " << j << ' ' << std::flush;
        for (int i = 0; i < image_width; ++i) {
            auto u = double(i) / (image_width-1);
            auto v = double(j) / (image_height-1);
            ray r(origin, lower_left_corner + u*horizontal + v*vertical - origin);
            color pixel_color = ray_color(r);
            write_color(std::cout, pixel_color);
        }
    }

    std::cerr << "\nDone.\n";
}
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711163803.png)

## 添加Sphere

### 光线与Sphere的交点
回一下初中数学 就是直线方程和模型方程的交点
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711164206.png)

判断b^2 - 4ac 的值是否大于0
```c++
bool hit_sphere(const point3& center, double radius, const ray& r) {
    vec3 oc = r.origin() - center;
    auto a = dot(r.direction(), r.direction());
    auto b = 2.0 * dot(oc, r.direction());
    auto c = dot(oc, oc) - radius*radius;
    auto discriminant = b*b - 4*a*c;
    return (discriminant > 0);
}

color ray_color(const ray& r) {
    if (hit_sphere(point3(0,0,-1), 0.5, r))
        return color(1, 0, 0);
    vec3 unit_direction = unit_vector(r.direction());
    auto t = 0.5*(unit_direction.y() + 1.0);
    return (1.0-t)*color(1.0, 1.0, 1.0) + t*color(0.5, 0.7, 1.0);
}
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711165323.png)

## 表面法线

（作者偏好）
首先，让我们计算表面法线，以便进行着色。表面法线是垂直于交点处表面的向量。在计算法线时，需要做两个设计决策。第一个是确定这些法线是否为单位长度。单位长度的法线在着色时很方便，所以我会选择是，但是我不会在代码中强制执行这一点。这可能会引入一些细小的错误，所以请注意这是个人偏好，就像大多数设计决策一样。对于一个球体而言，向外的法线方向是由击中点减去球心得到的：

```c++
double hit_sphere(const point3& center, double radius, const ray& r) {
    vec3 oc = r.origin() - center;
    auto a = dot(r.direction(), r.direction());
    auto b = 2.0 * dot(oc, r.direction());
    auto c = dot(oc, oc) - radius*radius;
    // 求根公式
    auto discriminant = b*b - 4*a*c;
    if (discriminant < 0) {
        return -1.0;
    } else {
        return (-b - sqrt(discriminant) ) / (2.0*a);
    }
}

color ray_color(const ray& r) {
    auto t = hit_sphere(point3(0,0,-1), 0.5, r);
    if (t > 0.0) {
        // 连接球心
        vec3 N = unit_vector(r.at(t) - vec3(0,0,-1));
        // 单位向量是 - 1   1， 转化为0 -- 1
        return 0.5*color(N.x()+1, N.y()+1, N.z()+1);
    }
    vec3 unit_direction = unit_vector(r.direction());
    t = 0.5*(unit_direction.y() + 1.0);
    return (1.0-t)*color(1.0, 1.0, 1.0) + t*color(0.5, 0.7, 1.0);
}
```

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711165827.png)

### 简化求根公式
```c++
double hit_sphere(const point3& center, double radius, const ray& r) {
    vec3 oc = r.origin() - center;
    auto a = dot(r.direction(), r.direction());
    auto b = 2.0 * dot(oc, r.direction());
    auto c = dot(oc, oc) - radius*radius;
    auto discriminant = b*b - 4*a*c;

    if (discriminant < 0) {
        return -1.0;
    } else {
        return (-b - sqrt(discriminant) ) / (2.0*a);
    }
}
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711165927.png)

```c++
double hit_sphere(const point3& center, double radius, const ray& r) {
    vec3 oc = r.origin() - center;
    auto a = r.direction().length_squared();
    auto half_b = dot(oc, r.direction());
    auto c = oc.length_squared() - radius*radius;
    auto discriminant = half_b*half_b - a*c;

    if (discriminant < 0) {
        return -1.0;
    } else {
        return (-half_b - sqrt(discriminant) ) / a;
    }
}
```

### Hittable Objects物体的抽象

为什么需要抽象？ 为了后续可以添加多个物体，以及多种不同类型的物体。他们有各自的hit方法
这个可碰撞的抽象类将有一个接受光线的碰撞函数。大多数光线追踪器发现为碰撞添加一个有效的区间 𝑡𝑚𝑖𝑛到 𝑡𝑚𝑎𝑥是很方便的，所以只有当 𝑡𝑚𝑖𝑛<𝑡<𝑡𝑚𝑎𝑥时，碰撞才“有效”。对于初始光线，这是正的 𝑡，但正如我们将看到的，为了在代码中处理一些细节，具有区间 𝑡𝑚𝑖𝑛到𝑡𝑚𝑎𝑥可以帮助我们。一个设计问题是，如果我们碰到了什么，是否要计算法线。随着搜索的进行，我们可能会碰到更近的物体，我们只需要最近物体的法线。我将选择简单的解决方案，计算一些我将存储在某个结构中的东西。以下是抽象类的示例：
```c++
struct hit_record {
    point3 p;
    vec3 normal;
    double t;
};

class hittable {
    public:
        virtual bool hit(const ray& r, double t_min, double t_max, hit_record& rec) const = 0;
};
```
修改sphere的函数
```c++
class sphere : public hittable {
    public:
        sphere() {}
        sphere(point3 cen, double r) : center(cen), radius(r) {};

        virtual bool hit(
            const ray& r, double t_min, double t_max, hit_record& rec) const override;

    public:
        point3 center;
        double radius;
};

bool sphere::hit(const ray& r, double t_min, double t_max, hit_record& rec) const {
    vec3 oc = r.origin() - center;
    auto a = r.direction().length_squared();
    auto half_b = dot(oc, r.direction());
    auto c = oc.length_squared() - radius*radius;

    auto discriminant = half_b*half_b - a*c;
    if (discriminant < 0) return false;
    auto sqrtd = sqrt(discriminant);

    // Find the nearest root that lies in the acceptable range.
    auto root = (-half_b - sqrtd) / a;
    if (root < t_min || t_max < root) {
        root = (-half_b + sqrtd) / a;
        if (root < t_min || t_max < root)
            return false;
    }

    rec.t = root;
    rec.p = r.at(rec.t);
    rec.normal = (rec.p - center) / radius;

    return true;
}
```

### Front Faces Versus Back Faces
判断正面还是反面
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711170535.png)
本文采用法线始终指向逆着射线的方向。如果射线在球体外部，法线将指向外部，但如果射线在球体内部，法线将指向内部。

```c++
struct hit_record {
    point3 p;
    vec3 normal;
    double t;
    bool front_face;

    inline void set_face_normal(const ray& r, const vec3& outward_normal) {
        front_face = dot(r.direction(), outward_normal) < 0;
        normal = front_face ? outward_normal :-outward_normal;
    }
};
```

修改main function
```c++
color ray_color(const ray& r, const hittable& world) {
    hit_record rec;
    if (world.hit(r, 0, infinity, rec)) {
        return 0.5 * (rec.normal + color(1,1,1));
    }
    vec3 unit_direction = unit_vector(r.direction());
    auto t = 0.5*(unit_direction.y() + 1.0);
    return (1.0-t)*color(1.0, 1.0, 1.0) + t*color(0.5, 0.7, 1.0);
}

int main() {

    // Image

    const auto aspect_ratio = 16.0 / 9.0;
    const int image_width = 400;
    const int image_height = static_cast<int>(image_width / aspect_ratio);

    // World
    hittable_list world;
    world.add(make_shared<sphere>(point3(0,0,-1), 0.5));
    world.add(make_shared<sphere>(point3(0,-100.5,-1), 100));

    // Camera

    auto viewport_height = 2.0;
    auto viewport_width = aspect_ratio * viewport_height;
    auto focal_length = 1.0;

    auto origin = point3(0, 0, 0);
    auto horizontal = vec3(viewport_width, 0, 0);
    auto vertical = vec3(0, viewport_height, 0);
    auto lower_left_corner = origin - horizontal/2 - vertical/2 - vec3(0, 0, focal_length);

    // Render

    std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";

    for (int j = image_height-1; j >= 0; --j) {
        std::cerr << "\rScanlines remaining: " << j << ' ' << std::flush;
        for (int i = 0; i < image_width; ++i) {
            auto u = double(i) / (image_width-1);
            auto v = double(j) / (image_height-1);
            ray r(origin, lower_left_corner + u*horizontal + v*vertical);
            color pixel_color = ray_color(r, world);
            write_color(std::cout, pixel_color);
        }
    }

    std::cerr << "\nDone.\n";
}
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711170914.png)

## Antialiasing(抗锯齿)
这个有太多资料了， games系列和opengl都有介绍 参考MSAA 高斯模糊
文章采用每个像素点，多次发射ray，然后平均采样
```c++
    for (int j = image_height-1; j >= 0; --j) {
        std::cerr << "\rScanlines remaining: " << j << ' ' << std::flush;
        for (int i = 0; i < image_width; ++i) {
            color pixel_color(0, 0, 0);
            for (int s = 0; s < samples_per_pixel; ++s) {
                auto u = (i + random_double()) / (image_width-1);
                auto v = (j + random_double()) / (image_height-1);
                ray r = cam.get_ray(u, v);
                pixel_color += ray_color(r, world);
            }
            write_color(std::cout, pixel_color, samples_per_pixel);
        }
    }
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711171216.png)

## Diffuse Materials 漫反射材质
不发光的漫反射物体仅仅采用其周围环境的颜色，但会通过自身固有的颜色对其进行调制。反射在漫反射表面上的光线会被随机化。因此，如果我们将三条光线发送到两个漫反射表面之间的缝隙中，它们每条光线都会有不同的随机行为：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711171417.png)

其实参考games101中提到的光照模型就可以。

在表面的击中点 𝑝 附近，有两个与之相切的单位半径球体。这两个球体的中心分别为 (𝐏+𝐧) 和 (𝐏−𝐧)，其中 𝐧 是表面的法线。以 (𝐏−𝐧) 为中心的球体被认为在表面内部，而以 (𝐏+𝐧) 为中心的球体被认为在表面外部。选择与射线起点在表面同侧的相切单位半径球体。在该单位半径球体内选择一个随机点 𝐒，并从击中点 𝐏 发送一条射线到随机点 𝐒（这是向量 (𝐒−𝐏) ）
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711171649.png)

```c++
vec3 random_in_unit_sphere() {
    while (true) {
        auto p = vec3::random(-1,1);
        if (p.length_squared() >= 1) continue;
        return p;
    }
}

color ray_color(const ray& r, const hittable& world) {
    hit_record rec;

    if (world.hit(r, 0, infinity, rec)) {
        point3 target = rec.p + rec.normal + random_in_unit_sphere();
        return 0.5 * ray_color(ray(rec.p, target - rec.p), world);
    }

    vec3 unit_direction = unit_vector(r.direction());
    auto t = 0.5*(unit_direction.y() + 1.0);
    return (1.0-t)*color(1.0, 1.0, 1.0) + t*color(0.5, 0.7, 1.0);
}
```

### 限制光线的弹射次数
```c++
color ray_color(const ray& r, const hittable& world, int depth) {
    hit_record rec;

    // If we've exceeded the ray bounce limit, no more light is gathered.
    if (depth <= 0)
        return color(0,0,0);

    if (world.hit(r, 0, infinity, rec)) {
        point3 target = rec.p + rec.normal + random_in_unit_sphere();
        return 0.5 * ray_color(ray(rec.p, target - rec.p), world, depth-1);
    }

    vec3 unit_direction = unit_vector(r.direction());
    auto t = 0.5*(unit_direction.y() + 1.0);
    return (1.0-t)*color(1.0, 1.0, 1.0) + t*color(0.5, 0.7, 1.0);
}
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711171843.png)

```c++
void write_color(std::ostream &out, color pixel_color, int samples_per_pixel) {
    auto r = pixel_color.x();
    auto g = pixel_color.y();
    auto b = pixel_color.z();

    // Divide the color by the number of samples and gamma-correct for gamma=2.0.
    auto scale = 1.0 / samples_per_pixel;
    r = sqrt(scale * r);
    g = sqrt(scale * g);
    b = sqrt(scale * b);

    // Write the translated [0,255] value of each color component.
    out << static_cast<int>(256 * clamp(r, 0.0, 0.999)) << ' '
        << static_cast<int>(256 * clamp(g, 0.0, 0.999)) << ' '
        << static_cast<int>(256 * clamp(b, 0.0, 0.999)) << '\n';
}
```

### Gamma 校正
可以参考之前发过的Gamma校正文章， 图像颜色更明亮点
```c++
void write_color(std::ostream &out, color pixel_color, int samples_per_pixel) {
    auto r = pixel_color.x();
    auto g = pixel_color.y();
    auto b = pixel_color.z();

    // Divide the color by the number of samples and gamma-correct for gamma=2.0.
    auto scale = 1.0 / samples_per_pixel;
    r = sqrt(scale * r);
    g = sqrt(scale * g);
    b = sqrt(scale * b);

    // Write the translated [0,255] value of each color component.
    out << static_cast<int>(256 * clamp(r, 0.0, 0.999)) << ' '
        << static_cast<int>(256 * clamp(g, 0.0, 0.999)) << ' '
        << static_cast<int>(256 * clamp(b, 0.0, 0.999)) << '\n';
}
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711172103.png)

### Lambertian Reflection
这里介绍的拒绝采样方法在单位球体上产生沿着表面法线偏移的随机点。这相当于在半球上选择方向，使靠近法线的概率较高，并且在接近切线角度时散射射线的概率较低。该分布按照 cos3(𝜙) 缩放，其中 𝜙 是与法线的角度。这很有用，因为以较小的角度到达的光线会在更大的区域上散布，因此对最终颜色的贡献较低。

然而，我们对Lambertian 分布感兴趣，它具有 cos(𝜙) 的分布。真正的Lambertian分布使射线在靠近法线处散射的概率更高，但分布更均匀。为了实现这一点，我们通过在单位球体的表面上选择随机点并沿着表面法线进行偏移来选择随机点。可以通过选择单位球体中的随机点，然后对其进行归一化来实现在单位球体上选择随机点
```c++
color ray_color(const ray& r, const hittable& world, int depth) {
    hit_record rec;

    // If we've exceeded the ray bounce limit, no more light is gathered.
    if (depth <= 0)
        return color(0,0,0);

    if (world.hit(r, 0.001, infinity, rec)) {
        point3 target = rec.p + rec.normal + random_unit_vector();
        return 0.5 * ray_color(ray(rec.p, target - rec.p), world, depth-1);
    }

    vec3 unit_direction = unit_vector(r.direction());
    auto t = 0.5*(unit_direction.y() + 1.0);
    return (1.0-t)*color(1.0, 1.0, 1.0) + t*color(0.5, 0.7, 1.0);
}
```

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711172259.png)

## Metal
对应光照模型中的高光。 回忆下光照模型 镜面反射+漫反射+环境光
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711172507.png)
反射方程

```c++
vec3 reflect(const vec3& v, const vec3& n) {
    return v - 2*dot(v,n)*n;
}
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711172639.png)

### 模糊反射
```c++
class metal : public material {
    public:
        metal(const color& a, double f) : albedo(a), fuzz(f < 1 ? f : 1) {}

        virtual bool scatter(
            const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered
        ) const override {
            vec3 reflected = reflect(unit_vector(r_in.direction()), rec.normal);
            scattered = ray(rec.p, reflected + fuzz*random_in_unit_sphere());
            attenuation = albedo;
            return (dot(scattered.direction(), rec.normal) > 0);
        }

    public:
        color albedo;
        double fuzz;
};
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711172809.png)

## Dielectrics
像水、玻璃和钻石等透明材料属于电介质。当光线射入这些材料时，它会分成反射光线和折射（透射）光线。我们将通过在反射和折射之间随机选择，并在每次相互作用中只生成一条散射光线来处理这个过程。
### Snell's Law
𝜂⋅sin𝜃=𝜂′⋅sin𝜃′
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711173011.png)

折射方程
```c++
vec3 refract(const vec3& uv, const vec3& n, double etai_over_etat) {
    auto cos_theta = fmin(dot(-uv, n), 1.0);
    vec3 r_out_perp =  etai_over_etat * (uv + cos_theta*n);
    vec3 r_out_parallel = -sqrt(fabs(1.0 - r_out_perp.length_squared())) * n;
    return r_out_perp + r_out_parallel;
}
```

### 全反射 Total Internal Reflection
当光线位于具有较高折射率的材料中时，Snell定律无法得到实际解，因此无法发生折射。回顾Snell定律和sin𝜃′的推导：
sin𝜃′=𝜂𝜂′⋅sin𝜃
如果光线在玻璃内部，外部是空气（𝜂=1.5和𝜂′=1.0）：

sin𝜃′=1.5*1.0⋅sin𝜃

sin𝜃′的值不能大于1。所以，如果满足以下不等式：
1.5*1.0⋅sin𝜃>1.0
两边的等式关系被打破，无法存在解。如果没有解存在，玻璃无法发生折射，因此必须反射光线：
```c++
if (refraction_ratio * sin_theta > 1.0) {
    // Must Reflect
    ...
} else {
    // Can Refract
    ...
}
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711173219.png)

### Schlick Approximation
想象下， 视线平行的看一个光滑的桌面和水面， 你看到的基本是物体的倒影
``` c++
    static double reflectance(double cosine, double ref_idx) {
        // Use Schlick's approximation for reflectance.
        auto r0 = (1-ref_idx) / (1+ref_idx);
        r0 = r0*r0;
        return r0 + (1-r0)*pow((1 - cosine),5);
    }
//======================================
     if (cannot_refract || reflectance(cos_theta, refraction_ratio) > random_double())
                direction = reflect(unit_direction, rec.normal);
            else
                direction = refract(unit_direction, rec.normal, refraction_ratio);

```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711173620.png)

## 构建相机矩阵
这个很基础， 可以参考games101中提到的
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711173744.png)

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711173817.png)

# 总结
传统的光线追踪， 光线与物体的交点 计算交点着色。
我们来画一幅完整的场景
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230711174032.png)
