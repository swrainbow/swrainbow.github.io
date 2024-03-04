---
title: css记录
date: 2022-11-20 22:02:08
tags: [css]
categories: [css]
---
## 水平垂直居中
1. 给父盒子设置属性
    - display： flex；
    - justify-content： center；
    - align-items： center
2. 给子盒子设置属性 
    - display:flex 子项设置 margin： auto
3. 绝对定位设置
    - 容器设置postition： relative 子元素设置position： absolute； left： 50%； top：50%； transform： translate（-50%， -50%）；
4. 绝对定位 资源色设置position: absolute; 设置固定宽度和高度；top、left、bottom、 right都为0， margin设置为auto

## 浏览器的渲染过程
浏览器收到HTML之后 会产生一个渲染任务， 并将其传递给渲染主线程的消息队列，在事件循环机制的作用下，渲染主线程会取出渲染任务，开启渲染流程。整个渲染流程分为多个阶段，
HTML解析，样式计算，布局，分层，绘制，分块，光栅化
1. 渲染第一步是解析HTML文档
2. 解析的过程中遇到HTML元素会解析HTML元素 最终生成dom树
3. 解析的过程中遇到style link 等css样式会解析成cssOM树 css不会阻塞HTML解析 Js会阻塞HTML解析
4. 根据DOm树， 依次为树中的每个节点计算出它最终的样式， 称之为computed Style。在这一过程中很多预设值会变成绝对值，比如red会变成rgb（255，0，0）；相对单位会变成绝对单位，em会变成px
5. 接下来是layout 节点的宽高
6. 分层layer 将来某一个曾改动后 仅会对该层进行后续处理 zindex transform opacity会影响分层结果
7. 绘制paint 完成绘制后，主线程将每个图层的信息交给合成线程，剩余工作由合成线程完成
8. 分块-》 光栅化-》 draw


## 什么是回流
1. reflow本质是重新计算layout树
当进行了会影响布局树的操作后，需要重新计算布局树， 引发layout
当js获取布局属性时 会立即reflow
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240220135313.png)
## 什么是重绘
repaint的本质就是重新根据分层信息计算了绘制指令
当改动了可见样式之后，就需要重新计算， 会引发repaint。
reflow一定会引起repaint
## 为什么transform的效率高
因为transofrm 既不影响布局也不会影响绘制指令， 它影响的只是渲染流程最后一个draw阶段。由于draw阶段在合成线程中，所以transform的变化几乎不会影响渲染主线程

## flex： 1
flex：flex-grow flex-shrink flex-basis
flex： 1 === flex： 1 1 auto

## 如何减少回流重绘
1. 尽量使用css属性简写 用border代替border-width， border-style， border-color
2. 批量修改元素样式
3. 避免使用table
4. 需要创建多个dom节点时，使用documentFragment创建
5. 尽量不要在for循环里面获取元素的位置或者大小属性， 利用好缓存

## 事件分级DOM0 DOM2
- DOM0
    - 所有的浏览器都支持，事件只能注册一次，后面的会覆盖旧的
- DOM2
    - DOM0和DOM2之间可以共存
    - el.addEventListener(type, listener, useCapture)
        - useCapture 是对象

## e.target 和 e.currentTarget
e.currentTarget始终是监听事件者， 即直接调用addEventlistener节点
e.target 是事件的真正出发着

## 浏览器的内核
navigator.appCodeName

## 为什么inline-block布局的时候存在一个空格的间距
换行引起的问题，需要给腹级设置font-size=0

## position 属性
1. relative
2. absolute
3. static
4. fixed
5. sticky

## 清除浮动
1. 绝对定位 他是脱离文档流
2. clear fix 浮动元素 会撑开父元素

## BFC
BFC 就是一个页面独立的容器， 容器里面的子元素不会影响到外面的元素
1. 浮动元素
2. 绝对定位
3. overflow 除了visible
4. display： flex ？ inline-block table-cell

## css中的伪元素
- ::before  在元素内容前面插入
- :: after
- :: first-letter

## background-size 属性有哪些
- auto 默认值 原始尺寸
- <length> : 可以使用长度单位px em y
- <percentage> 可以使用百分比， 相对于背景区域
- cover 将背景图片缩放，完全覆盖  图片会被裁剪
- contain 宽高比不变，背景区域留白 完全适应背景区域

## position: fixed
当一个元素包含fixed 属性时，屏幕视口（viewport）会为其创建一个包含块（containing block），其大小就是 viewport 的大小，然后该 fixed 元素基于该包含块进行定位。所以通常我们会说 fixed 元素是相对 viewport 来定位的。
此外，fixed 属性会创建新的层叠上下文。当元素祖先的 transform, perspective 或 filter 属性非 none 时，容器由视口改为该祖先。
