---
title: vite-1
date: 2022-12-10 21:17:14
tags: [vite]
categories: [vite]
---
# How vite and other work 这类打包工具如何工作

1. 需要把编写的类javascript文件转化成 js，浏览器能运行的是js
    - 去除类型标注
    - vue/jsx 到 js的转化
    - 将“node_modules”导入转换为带有扩展名的相对路径
2. 需要一个good 开发体验
    - HMR
    - Source maps
3. 生产环境的结果
    - the bundle should be small remove unused code /Tree shaking

## vite alias
```js
// vite.config.ts
import path from 'path'; {
resolve: {
// 别名配置 alias: {
      '@assets': path.join(__dirname, 'src/assets')
    }
} 
```
## Web Worker 脚本
```js
import Worker from './example.js?worker'; // 1. 初始化 Worker 实例
const worker = new Worker();
// 2. 主线程监听 worker 的信息 worker.addEventListener('message', (e) => {
  console.log(e);
});

const start = () => { let count = 0; setInterval(() => {
// 给主线程传值
    postMessage(++count);
  }, 2000);
}; start();
```

## svg 组件加载方式
- Vue2 项目中可以使用 vite-plugin-vue2-svg插件。 
- Vue3 项目中可以引入 vite-svg-loader。


# 为什么需要预构建
- vite 是基于浏览器原生ES模块规范实现的Dev Server， 不论是应用代码，还是第三方依赖的代码，理应符合ESM规范。但是第三方打包规范，还有相当多的第三方库没有ES版本。所以我们需要将它转化为ESM格式
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240124111033.png)
- 请求瀑布流问题，loadsh-es 库本 身是有 ES 版本产物的，可以在 Vite 中直接运行。但实际上，它在加载时会发出特别多 的请求，导致页面加载的前几秒几都乎处于卡顿状态。在进行依赖的预构建之后， lodash-es 这个库的代码被打包成了一 个文件，这样请求的数量会骤然减少，页面加载也快了许多
- vite 一个import 就是一个http请求

## vite基石 Rollup的插件机制
- 仅仅使用 Rollup 内置的打包能力很难满足项目日益复 杂的构建需求。对于一个真实的项目构建场景来说，我们还需要考虑到 模块打包 之外的 问题，比如路径别名(alias) 、全局变量注入和代码压缩
- Rollup 的打包过程中，会定义一套完整的构建生命周期，从开始打包到产物输出，中途 会经历一些标志性的阶段，并且在不同阶段会自动执行对应的插件钩子函数(Hook)。对 Rollup 插件来讲，最重要的部分是钩子函数，一方面它定义了插件的执行逻辑，也就 是"做什么";另一方面也声明了插件的作用阶段，即"什么时候做"，这与 Rollup 本身的 构建生命周期息息相关。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240125113115.png)
- 对于一次完整的构建过程而言， Rollup 会先进入到 Build 阶段，解析各模块的内 容及依赖关系，然后进入 Output 阶段，完成打包及输出的过程


一是将其他格式(如 UMD 和 CommonJS)的产物转换为 ESM 格式，使其在浏览器通过 <script type="module"><script> 的方式正常加载。 二是打包第三方库的代码，将各个第三方库分散的文件合并到一起，减少 HTTP 请求数
量，避免页面加载性能劣化。

而这两件事情全部由性能优异的 Esbuild (基于 Golang 开发)完成，而不是传统的 Webpack/Rollup，所以也不会有明显的打包性能问题，反而是 Vite 项目启动飞快(秒级 启动)的一个核心原因。

## vite架构图
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240311101358.png)

双引擎： esbuild 和 rollup
### Esbuild 到底在 Vite 的构建体系中发挥了哪些作用
1. 首先是开发阶段的依赖预构建阶段
    - 主要是 ESM 格式的兼容性问题和海量请求的 问题

Vite 1.x 版本中使用 Rollup 来做这件事情，但 Esbuild 的性能实在是太恐怖了，Vite 2.x 果断采用 Esbuild 来完成第三方依赖的预构建，至于性能到底有多强，大家可以参照它与 传统打包工具的性能对比图:
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240311101758.png)

Esbuild 作为打包工具也有一些缺点
- 不支持降级到 ES5 的代码。这意味着在低端浏览器代码会跑不起来。
不支持 const enum 等语法。这意味着单独使用这些语法在 esbuild 中会直接抛错。
- 不提供操作打包产物的接口，像 Rollup 中灵活处理打包产物的能力(如 renderChunk 钩子)在 Esbuild 当中完全没有。
- 不支持自定义 Code Splitting 策略。传统的 Webpack 和 Rollup 都提供了自定义拆 包策略的 API，而 Esbuild 并未提供，从而降级了拆包优化的灵活性。

2. 单文件编译——作为 TS 和 JSX 编译工具
vite 使用 Esbuild 进行语法转译，也就是将 Esbuild 作为 Transformer 来用。 大家可以在架构图中 Vite Plugin Pipeline 部分注意到:
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240311102135.png)

- 缺点：
这是因为 Esbuild 并没有实现 TS 的类型系 统，在编译 TS (或者 TSX ) 文件时仅仅抹掉了类型相关的代码，暂时没有能力实现类型 检查。

3. 代码压缩——作为压缩工具
从架构图中可以看到，在生产环境中 Esbuild 压缩器通过插件的形式融入到了 Rollup 的 打包流程中:
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240311102600.png)
传统的方式都是使用 Terser 这种 JS 开发的压缩器来实现，在 Webpack 或者 Rollup 中 作为一个 Plugin 来完成代码打包后的压缩混淆的工作。但 Terser 其实很慢，主要有 2 个原因。
压缩这项工作涉及大量 AST 操作，并且在传统的构建流程中，AST 在各个工具之间 无法共享，比如 Terser 就无法与 Babel 共享同一个 AST，造成了很多重复解析的过 程。
JS 本身属于解释性 + JIT(即时编译) 的语言，对于压缩这种 CPU 密集型的工作， 其性能远远比不上 Golang 这种原生语言。

## vite 构建基石 Rollup
Rollup 在 Vite 中的重要性一点也不亚于 Esbuild，它既是 Vite 用作生产环境打包的核心 工具，也直接决定了 Vite 插件机制的设计。

自动预加载。Vite 会自动为入口 chunk 的依赖自动生成预加载标签 rel="moduelpreload"> 


兼容插件机制
无论是开发阶段还是生产环境，Vite 都根植于 Rollup 的插件机制和生态

# Esbuild 功能使用

## why esbuild fast？
- 使用 Golang 开发，构建逻辑代码直接被编译为原生机器码，而不用像 JS 一样先代 码解析为字节码，然后转换为机器码，大大节省了
- 多核并行
- 从零造轮子。 几乎没有使用任何第三方库，所有逻辑自己编写，大到 AST 解析，小 到字符串的操作，保证极致的代码性能。
- 高效的内存利用。Esbuild 中从头到尾尽可能地复用一份 AST 节点数据，而不用像 JS 打包工具中频繁地解析和传递 AST 数据(如 string -> TS -> JS -> string)，造 成内存的大量浪费。

## 代码调用
Esbuild 对外暴露了一系列的 API，主要包括两类: Build API 和 Transform API 

Esbuild 插件结构被设计为一个对象，里面有 name 和 setup 两个属性， name 是插件的 名称， setup 是一个函数，其中入参是一个 build 对象，这个对象上挂载了一些钩子可 供我们自定义一些钩子函数逻辑。以下是一个简单的 Esbuild 插件示例:
```js
let envPlugin = { name: 'env', setup(build) {
    build.onResolve({ filter: /^env$/ }, args => ({
      path: args.path,
      namespace: 'env-ns',
}))
    build.onLoad({ filter: /.*/, namespace: 'env-ns' }, () => ({
      contents: JSON.stringify(process.env),
      loader: 'json',
})) },
}
require('esbuild').build({
  entryPoints: ['src/index.jsx'],
bundle: true, outfile: 'out.js',
// 应用插件
plugins: [envPlugin],
}).catch(() => process.exit(1))


// 应用了 env 插件后，构建时将会被替换成 process.env 对象
import { PATH } from 'env'
console.log(`PATH is ${PATH}`)
```


## onResolve 钩子 和 onLoad 钩子
这两个钩子函数中都需要传入两个参数: Options 和 Callback

Options 。它是一个对象，对于 onResolve 和 onload 都一样，包含 filter 和 namespace 两个属性，类型定义如下：
filter 为必传参数， 是一个正则比奥大使，决定了要过滤出的特征文件。

namespace为选填参数，一般在 钩子中的回调参数返回 属性作为 标识，我们可以在 钩子中通过 将模块过滤出来。如上述插件示例就 在 钩子通过 这个 namespace 标识过滤出了要处理的 env 模块。

## 其他钩子
onStart 和 onEnd 两个钩子用来在构 建开启和结束时执行一些自定义的逻辑
```js
let examplePlugin = { name: 'example',
setup(build) { build.onStart(() => {
      console.log('build started')
    });
build.onEnd((buildResult) => {
if (buildResult.errors.length) {
return; }
// 构建元信息
// 获取元信息后做一些自定义的事情，比如生成 HTML console.log(buildResult.metafile)
}) },
}
```
## Rollup打包概念
Rollup 是一款基于 ES Module 模块规范实现的 JavaScript 打包工具，在前端社区中赫 赫有名，同时也在 Vite 的架构体系中发挥着重要作用。不仅是 Vite 生产环境下的打包工 具，其插件机制也被 Vite 所兼容，可以说是 Vite 的构建基石

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240311130718.png)
1. 首先经历optins钩子进行配置的转换，得到处理后的配置对象。
2. 随之 Rollup 会调用 buildStart钩子， 正式开始流程
3. Rollup 先进入到resolveId钩子中解析文件路径开始 从input配置中制定的入口文件开始

4. Rollup通过调用load钩子加载模块内容。

5. 紧接着 Rollup 执行所有的transform钩子来对模块内容进行进行自定义的转换,比如 babel 转译。
6. 现在 Rollup 拿到最后的模块内容，进行 AST 分析，得到所有的 import 内容，调用 moduleParsed 钩子:
  - 如果是普通的 import，则执行 钩子，继续回到步骤 3
  - 如果是动态 import，则执行 钩子解析路径，如果
解析成功，则回到步骤 4 加载模块，否则回到步骤 3 通过 解析路 径。

7. 直到所有的 import 都解析完毕，Rollup 执行 钩子，Build 阶段结束

# 编写一个vite 插件
```js
// myPlugin.js 推荐以 vite-plugin 开 头
export function myVitePlugin(options) { console.log(options)
return {
name: 'vite-plugin-xxx', load(id) {
// 在钩子逻辑中可以通过闭包访问外部的 options 传参 }
} }

// 使用方式
// vite.config.ts
import { myVitePlugin } from './myVitePlugin'; export default {
plugins: [myVitePlugin({ /* 给插件传参 */ })] }
```

vite 会调用一系列与rollup兼容的钩子， 主要分为三个阶段
- 服务器启动阶段： optins和 buildStart钩子会在服务启动时被调用
- 请求响应阶段：当浏览器发起请求时， Vite内部一次调用resolveId、load和transform钩子
- 服务器关闭阶段 Vite会一次执行buildEnd和closeBundle钩子

## vite 独有的钩子函数

- config : 用来进一步修改配置。
- configResolved : 用来记录最终的配置信息。
- configureServer : 用来获取 Vite Dev Server 实例，添加中间件。
- transformIndexHtml : 用来转换 HTML 的内容。
- handleHotUpdate : 用来进行热更新模块的过滤，或者进行自定义的热更新处理。

## 插件Hook执行顺序

![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240311175445.png)
- 服务启动阶段: config 、 configResolved 、 options 、 configureServer 、 buildStart
- 请求响应阶段: 如果是 html 文件，仅执行 transformIndexHtml 钩子;对于非 HTML 文件，则依次执行 resolveId 、 load 和 transform 钩子。相信大家学过 Rollup 的插 件机制，已经对这三个钩子比较熟悉了。
- 热更新阶段: 执行 handleHotUpdate 钩子。
- 服务关闭阶段: 依次执行 buildEnd 和 closeBundle 钩子。

Vite中插件执行顺序
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240311175640.png)

## svg 插件


# 拆包 Code Splitting
- bundle指的是整体的打包产物，包含 JS 和各种静态资源。 
- chunk指的是打包后的 JS 文件，是bundle的子集。
- vendor是指第三方包的打包产物，是一种特殊的 chunk。

## why?
- 无法做到按需加载，即使是当前页面不需要的代码也会进行加载。
- 线上缓存复用率极低，改动一行代码即可导致整个 bundle 产物缓存失效。
```
├── assets
│   ├── Dynamic.3df51f7a.js
│   ├── Dynamic.f2cbf023.css
│   ├── favicon.17e50649.svg
│   ├── index.1e236845.css
│   ├── index.6773c114.js
│   └── vendor.ab4b9e1f.js
└── index.html
// Async Chunk
// Async Chunk (CSS) // 静态资源
// Initial Chunk (CSS) // Initial Chunk
// 第三方包产物 Chunk // 入口 HTML
```
Vite 实现了自动 CSS 代码分割的能力，即实现一个 chunk 对应一个 css 文件， 比如上面产物中 index.js 对应一份 index.css,而按需加载的 chunk Danamic.js 也对
应单独的一份 Danamic.css 文件，与 JS 文件的代码分割同理，这样做也能提升 CSS 文件 的缓存复用率。

- 对于 Initital Chunk 而言，业务代码和第三方包代码分别打包为单独的 chunk，在 上述的例子中分别对应 index.js 和 vendor.js 。需要说明的是，这是 Vite 2.9 版本 之前的做法，而在 Vite 2.9 及以后的版本，默认打包策略更加简单粗暴，将所有的 js 代码全部打包到 index.js 中。
- 对于 Async Chunk 而言 ，动态 import 的代码会被拆分成单独的 chunk，如上述的 Dynacmic 组件。

## 自定义拆包

```js
// vite.config.ts
{
build: {
    rollupOptions: {
      output: {
          // manualChunks 配置 manualChunks: {
          'lodash': ['lodash-es'],
          // 将组件库的代码打包
          'library': ['antd', '@arco-design/web-react'],
        },
      }, 
    }
  }, 
}
```
在对象格式的配置中， key 代表 chunk 的名称， value 为一个字符串数组，每一项为第 三方包的包名
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240312095549.png)
原来的 vendor 大文件被拆分成了我们手动指定的几个小 chunk，每个 chunk 大概 200 KB 左右，是一个比较理想的 chunk 体积。这样，当第三方包更新的时 候，也只会更新其中一个 chunk 的 url，而不会全量更新，从而提高了第三方包产物的缓 存命中率。

### vite-plugin-chunk-split
```js
// vite.config.ts
import { chunkSplitPlugin } from 'vite-plugin-chunk-split';
export default { chunkSplitPlugin({
  // 指定拆包策略 
    customSplitting: {
    // 1. 支持填包名。`react` 和 `react-dom` 会被打包到一个名为`render-vendor`的 chunk 里面
    'react-vendor': ['react', 'react-dom'],
    // 2. 支持填正则表达式。src 中 components 和 utils 下的所有文件被会被打包为`component-util` 
    'components-util': [/src\/components/, /src\/utils/]
  } 
  })
}

```

# 语法降级 与 polyfill
## 工具概览
- 编译时工具  @babel/preset-env 和 @babel/plugin-transform-runtime
- 运行时基础库 代表库包括core-js 和 regenerator-runtime

编译时工具的作用是在代码编译阶段进行语法降级及添加

运行时基础库是根据 ESMAScript 官方语言规范提供各种 Polyfill 实现代码，主要包 括 core-js 和 regenerator-runtime 两个基础库，不过在 babel 中也会有一些上层的封装

## vite语法降级与Polyfill注入
@vitejs/plugin-legacy

我们可以 基于它来解决项目语法的浏览器兼容问题。这个插件内部同样使用 @babel/preset-env
以及corejs等一系列基础库来进行语法降级和 Polyfill 注入

```js
// vite.config.ts
import legacy from '@vitejs/plugin-legacy'; import { defineConfig } from 'vite'
export default defineConfig({ plugins: [
// 省略其它插件 
legacy({
// 设置目标浏览器，browserslist 配置语法
      targets: ['ie >= 11'],
    })
] })
```
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240312143536.png)
通过插件， Vite 会分别打包出modern模式和Legacy模式的产物，将两种产物插入同一个 HTML 里面， Modern产物被放到 type = “module“的 script 标签， Legacy放到带有nomodule的script标签
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240312143733.png)
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240312143907.png)