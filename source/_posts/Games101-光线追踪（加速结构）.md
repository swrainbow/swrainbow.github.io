---
title: Games101-光线追踪（加速结构）
date: 2023-07-05 20:12:39
tags: [图形学]
categories: [games101 系列]
---
Whitted Style光追中，判断光线与场景相交需要去与所有三角形求交，渲染问题过慢。
因此这节找到一些加速的办法。
# 轴对齐包围盒
Bouding Volumes包围盒用于快速判断光线和物体是否有交集。
包围盒完全包围物体，如果光线没有打到包围盒，那么光线一定和物体没有交集。所以先测试包围盒，再测试物体。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705201347.png)
将将长方体认为是三个不同的对面（pair of slabs）形成的交集，得到
AABB包围盒：由三对平面的交集构成，任意一对平面都与x-axis, y-axis, 或者z-axis垂直，所以称之为轴对齐包围盒。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705201412.png)
下面讨论光线和AABB包围盒的求交。
2D AABB情况，此时只有x ,y两对平面
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705201432.png)
分别求出进入包围盒一对对面时间$t_{min}$和离开时间$t_{max}$，则两个$[t_{min},t_{max}]$的交集为在包围盒里的时间段。
如果任意的 t < 0，那么这代表的是光线反向传播与对应平面的交点，在下面进行讨论。
扩展到3D AABB情况，首先确定进入或离开包围盒的条件：
- 进入包围盒：进入所有对面
- 离开包围盒：离开任一对面

得到进入和离开的时间：
$\large t_{enter} = max\{t_{min}\},~t_{exit}=min\{t_{max}\}(t_{exit}>=0)$
讨论限制条件：
- $t_{exit}<0$ 
  - 包围盒在光线背后——没有交点
- $t_{exit}>=0\&\&t_{enter}<0$ 
  - 原点在包围盒内部——有交点
- 得到光线与AABB包围盒有交点当且仅当$\large t_{enter}<t_{exit}\&\&e_{exit}>=0$

AABB包围盒带来的计算效率的提升：

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705201517.png)

AABB包围盒的缺点：
- 整个场景只有一个极其复杂的单一人物模型，那么只对这一个物体做包围盒的话，相当于对效率没有任何提升
- 整个场景充斥着大量的细小模型，如草，花之类的，每个模型可能只有很少的面，如果此时对每个物体求包围盒，得到的包围盒数量会相当之多，对于光线追踪效率来说效率提升有限
# 均匀格子划分
均匀格子划分预处理出许多个均匀的小的AABB包围盒，再根据光线方向找出相交方格。再判断方格内是否存储物体信息，若有则进一步求交。

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705201551.png)

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705201602.png)
这种划分方法假设了找出相交方格要比直接判断与物体求交相对容易，因此划分方格数的多少也是性能的关键，方格太少，没有加速效果，方格太多，判断与方格的求交可能会拖累效率。
一般来说，取格子数量为27。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705201619.png)

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705201634.png)

# 空间划分
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705201652.png)

- Oct-tree 
  - 八叉树
  - 维度随维数指数级增长
- KD-Tree 
  - 每次沿着一个轴方向划分
- BSP-Tree 
  - 二分
  - 每次取一个方向
  - 困难

## KD-Tree空间划分
KD-Tree，每次将空间划分为两部分，且划分依次沿着x-axis，y-axis，z-axis，第一次横着将2维空间分为上下，第二次再竖着将上下两个子空间分别划分为左右部分，依次递归划分，终止条件与八叉树类似。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705201805.png)
划分规定：
- 依次沿着x-axis,y-axis,z-axis划分，使得空间被划分的更加平衡
- 划分的位置由空间中三角面的分布决定
- 叶子节点存储对应空间的所有物体或三角面信息，中间节点仅存储指针指向两个子空间
- 当划分空间太小或是子空间内只有少量三角形则停止划分
- 叶子结点存储物体信息

与光线求交点：从树根DFS递归判断，注意存储物体信息的是叶子结点。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705201855.png)
注意：图中做了简化，最大包围盒的左半边实际上也要进行划分，所以左半部分对应的1号空间是叶子节点。如果光线与之相交，进一步判断与存储与叶子节点的物体信息求交。左半边判断完之后，接着判断右半部分。
优点：
- 用KD-Tree的结构来构建AABB的好处是倘若光线与哪一部分空间不相交，那么则可以省略该部分空间所有子空间的判断过程，在原始的纯粹的AABB之上更进一步提升了加速效率。

缺点：
- 一个物体可能出现在不同的叶子节点
- 要考虑三角形和盒子求交（难）


- Bouding Volume Hierarchy(BVH)
BVH不以空间作为划分依据，而是从物体的角度考虑，即三角形面。
划分的步骤与KD-Tree类似。

- 找到bounding box
- 拆分成两个结点
- 重新计算bounding box
- 出口条件
- 把物体存在叶子结点

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705201936.png)
注意，这里右半部分实际上也应该进行划分。

- 一个物体只会出现在一个bounding box
- 不用三角形求交 直接bouding box

引起的问题：空间不严格划分
解决方法：合理划分结点
- 每次划分一般选择最长的那一轴划分
- 选择这个轴上的中位数进行划分 
  - 快速选择算法

伪代码：
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230705202011.png)