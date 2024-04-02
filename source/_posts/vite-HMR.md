---
title: vite-模块联邦 & 其他
date: 2023-12-06 19:12:27
tags: [vite]
categories: [vite]
---
# 模块联邦 
## 模块共享之痛
对于一个互联网产品来说，一般会有不同的细分应用，比如同花顺分为A股、港美股、东南亚等
站点，而每个子站又彼此独立，可能由不同的开发团队进行单独的开发和维护，看似没有
什么问题，但实际上会经常遇到一些模块共享的问题，也就是说不同应用中总会有一些共
享的代码，比如公共组件、公共工具函数、公共第三方依赖等等。对于这些共享的代码，
除了通过简单的复制粘贴，还有没有更好的复用手段?

1. 发布npm 包
发布 npm 包是一种常见的复用模块的做法，我们可以将一些公用的代码封装为一个 npm 包，具体的发布更新流程是这样的:
公共库 lib1 改动，发布到 npm; 所有的应用安装新的依赖，并进行联调。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240312170341.png)
封装 npm 包可以解决模块复用的问题，但它本身又引入了新的问题:
开发效率问题。每次改动都需要发版，并所有相关的应用安装新依赖，流程比较复 杂。 项目构建问题。引入了公共库之后，公共库的代码都需要打包到项目最后的产物 后，导致产物体积偏大，构建速度相对较慢。
因此，这种方案并不能作为最终方案，只是暂时用来解决问题的无奈之举
2. 依赖外部化(external)+ CDN 引入
兼容性问题。并不是所有的依赖都有 UMD 格式的产物，因此这种方案不能覆盖所有的第三方 npm 包。

依赖顺序问题。通常需要考虑间接依赖的问题，如对于antd 组件库，它本身也依赖了 react 和 moment，那么 react 和 moment 也需要 external ，并且在 HTML 中引用这些包，同时也要严格保证引用的顺序，比如说 moment 如果放在了antd 后面，代码可能无法运行。而第三方包背后的间接依赖数量一般很庞大，如果 逐个处理，对于开发者来说简直就是噩梦。

产物体积问题。由于依赖包被声明 external 之后，应用在引用其 CDN 地址时，会 全量引用依赖的代码，这种情况下就没有办法通过 Tree Shaking 来去除无用代码了，会导致应用的性能有所下降。
3. Monorepo
所有的应用代码必须放到同一个仓库。如果是旧有项目，并且每个应用使用一个 Git 仓库的情况，那么使用 Monorepo 之后项目架构调整会比较大，也就是说改造成本 会相对比较高。

Monorepo 本身也存在一些天然的局限性，如项目数量多起来之后依赖安装时间会 很久、项目整体构建时间会变长等等，我们也需要去解决这些局限性所带来的的开 发效率问题。而这项工作一般需要投入专业的人去解决，如果没有足够的人员投入 或者基建的保证，Monorepo 可能并不是一个很好的选择。

项目构建问题。跟发npm 包的方案一样，所有的公共代码都需要进入项目的构建流程中，产物体积还是会偏大

## MF 核心概念
模块联邦中主要有两种模块: 本地模块和远程模块 。

本地模块即为普通模块，是当前构建流程中的一部分，而远程模块不属于当前构建流程， 在本地模块的运行时进行导入，同时本地模块和远程模块可以共享某些依赖的代码。
值得强调的是，在模块联邦中，每个模块既可以是 本地模块 ，导入其它的 远程模块 ，又 可以作为远程模块，被其他的模块导入。如下面这个例子所示:
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240312170737.png)

## 应用
使用pnpm install @originjs/vite-plugin-federation -D 安装插件
在配置文件中添加配置

```js
// 远程模块配置
// remote/vite.config.ts
import { defineConfig } from "vite";
import vue from "@vitejs/plugin-vue";
import federation from "@originjs/vite-plugin-federation";
// https://vitejs.dev/config/
export default defineConfig({ plugins: [
        vue(),
        // 模块联邦配置 
        federation({
        name: "remote_app", 
        filename: "remoteEntry.js", // 导出模块声明
        exposes: {
                "./Button": "./src/components/Button.js",
                "./App": "./src/App.vue",
                "./utils": "./src/utils.ts",
        },
        // 共享依赖声明
            shared: ["vue"],
            }),
        ],
        // 打包配置 build: 
        {
            target: "esnext",
        },
});

// 本地模块配置
// host/vite.config.ts
import { defineConfig } from "vite";
import vue from "@vitejs/plugin-vue";
import federation from "@originjs/vite-plugin-federation";
export default defineConfig(
    { 
        plugins: [
        vue(),
        federation({
              // 远程模块声明 
            remotes: {
                    remote_app: "http://localhost:3001/assets/remoteEntry.js",
                },
            // 共享依赖声明
            shared: ["vue"],
        }),
        ], 
        build: {
            target: "esnext",
        },
});

```
- 远程模块通过 exposes 注册导出的模块，本地模块通过 remotes 注册远程模块 地址。
远程模块进行构建，并部署到云端。
- 本地通过 import '远程模块名称/xxx' 的方式来引入远程模块，实现运行时加载。

## MF实现原理
```js
const remotesMap = {
    'remote_app':{url:'http://localhost:3001/assets/remoteEntry.js',format:'esm',from:'vite'},
    'shared':{url:'vue',format:'esm',from:'vite'}
  };
async function ensure() {
const remote = remoteMap[remoteId]; // 做一些初始化逻辑，暂时忽略
// 返回的是运行时容器
}
async function getRemote(remoteName, componentName) { return ensure(remoteName)
// 从运行时容器里面获取远程模块
.then(remote => remote.get(componentName)) .then(factory => factory());
}
// import 语句被编译成了这样
// tip: es2020 产物语法已经支持顶层 await
const __remote_appApp = await getRemote("remote_app" , "./App");
```
除了 import 语句被编译之外，在代码中还添加了 remoteMap 和一些工具函数，它们的目 的很简单，就是通过访问远端的运行时容器来拉取对应名称的模块。
而运行时容器其实就是指远程模块打包产物 remoteEntry.js 的导出对象
```js
// remoteEntry.js
const moduleMap = { "./Button": () => {
return import('./__federation_expose_Button.js').then(module => () => module) },
"./App": () => {
dynamicLoadingCss('./__federation_expose_App.css');
return import('./__federation_expose_App.js').then(module => () => module);
  },
  './utils': () => {
return import('./__federation_expose_Utils.js').then(module => () => module); }
};
// 加载 css
const dynamicLoadingCss = (cssFilePath) => {
    const metaUrl = import.meta.url;
    if (typeof metaUrl == 'undefined') {
        console.warn('The remote style takes effect only when the build.target option in the vite')
        return
    }
    const curUrl = metaUrl.substring(0, metaUrl.lastIndexOf('remoteEntry.js')); const element = document.head.appendChild(document.createElement('link')); element.href = curUrl + cssFilePath;
    element.rel = 'stylesheet';
};
// 关键方法，暴露模块 
const get =(module) => {
    return moduleMap[module](); 
};
const init = () => {
// 初始化逻辑，用于共享模块，暂时省略
}
export { dynamicLoadingCss, get, init }
```
moduleMap 用来记录导出模块的信息，所有在 exposes 参数中声明的模块都会打包成单独的文件，然后通过 dynamic import 进行导入。 

容器导出了十分关键的 get 方法，让本地模块能够通过调用这个方法来访问到该
远程模块。
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20240312172745.png)

# Pure Esm
modulepreload 。以前我们会在link标签中加上 rel="preload" 来进行资源预加载，即在浏览器解析HTML之前就开始加载资源，现在对于ESM也有对应的
modulepreload来支持这个行为。

JSON Modules 和 CSS Modules ，即通过如下方式来引入 json 或者 css :
```html
<script type="module">
// 获取 json 对象
import json from 'https://site.com/data.json' assert { type: 'json' }; // 获取 CSS Modules 对象
import sheet from 'https://site.com/sheet.css' assert { type: 'css' }; 
</script>
```

# 如何体系化地对 Vite 项目进行性能优化
- 网络优化。包括 HTTP2 、 DNS 预解析 、 Preload 、 Prefetch 等手段。 
- 资源优化。包括 构建产物分析 、 资源压缩 、 产物拆包 、 按需加载 等优化方式。
- 预渲染优化，本文主要介绍 服务端渲染 (SSR)和 静态站点生成 (SSG)两种手段。 
- 客户端离线包

在 HTTP 1.1 协议中，队头阻塞和请求排队问题很容易成为网络层的性能瓶颈。而 HTTP 2 的诞生就是为了解决这些问题，它主要实现了如下的能力:
多路复用。将数据分为多个二进制帧，多个请求和响应的数据帧在同一个 TCP 通道 进行传输，解决了之前的队头阻塞问题。而与此同时，在 HTTP2 协议下，浏览器不 再有同域名的并发请求数量限制，因此请求排队问题也得到了解决。
Server Push，即服务端推送能力。可以让某些资源能够提前到达浏览器，比如对于 一个 html 的请求，通过 HTTP 2 我们可以同时将相应的 js 和 css 资源推送到浏览 器，省去了后续请求的开销。
## Preload/Prefetch
对于一些比较重要的资源，我们可以通过 Preload 方式进行预加载，即在资源使用之前 就进行加载，而不是在用到的时候才进行加载，这样可以使资源更早地到达浏览器

在 Vite 中我们可以通过配置一键开
启 modulepreload 的 Polyfill，从而在使所有支持原生 ESM 的浏览器(占比 90% 以上)都 能使用该特性，配置方式如下:
```js
// vite.config.ts
export default { 
    build: {
      polyfillModulePreload: true
    }
}
```
Prefetch 也是一个比较常用的优化方式，它相当于告诉浏览器空闲的 时候去预加载其它页面的资源

# 依赖预构建 深入

1. 首先会有缓存判断，Vite 在每次预构建之后都将一些关键信息写入到了 _metadata.json 文件中，第二次启动项目时会通过这个文件中的 hash 值来进行缓存的
判断，如果命中缓存则不会进行后续的预构建流程
2. 依赖扫描
如果没有命中缓存，则会正式地进入依赖预构建阶段。不过 Vite 不会直接进行依赖的预构建，而是在之前探测一下项目中存在哪些依赖，收集依赖列表，也就是进行依赖扫描的过程。这个过程是必须的，因为Esbuild需要知道我们到底要打包哪些第三方依赖。关键代码如下:
```js
const deps: Record<string, string> = {};
// 扫描用到的 Esbuild 插件
    const plugin = esbuildScanPlugin(config, container, deps, missing, entries); 
    await Promise.all(
    // 应用项目入口 
        entries.map((entry) =>
            build({
                absWorkingDir: process.cwd(), // 注意这个参数
                write: false,
                entryPoints: [entry],
                bundle: true,
                format: "esm",
                logLevel: "error",
                plugins: [...plugins, plugin], ...esbuildOptions,
            }) 
    )
);

```
值得注意的是，其中传入的write参数被设为false，表示产物不用写入磁盘，这就大大节省了磁盘 I/O 的时间了，也是 依赖扫描为什么往往比依赖打包快很多的原因之一。
3. 依赖打包
调用 Esbuild 进行打包并 写入产物到磁盘中，关键代码如下:
```js
const result = await build({
    absWorkingDir: process.cwd(),
    // 所有依赖的 id 数组，在插件中会转换为真实的路径 entryPoints: Object.keys(flatIdDeps),
    bundle: true,
    format: "esm",
    target: config.build.target || undefined, external: config.optimizeDeps?.exclude, logLevel: "error",
    splitting: true,
    sourcemap: true,
    outdir: cacheDir,
    ignoreAnnotations: true,
    metafile: true,
    define,
    plugins: [
    ...plugins,
    // 预构建专用的插件
        esbuildDepPlugin(flatIdDeps, flatIdToExports, config, ssr),
    ],
    ...esbuildOptions,
    });
// 打包元信息，后续会根据这份信息生成 _metadata.json 
const meta = result.metafile!;
```
在打包过程完成之后，Vite 会拿到 Esbuild 构建的元信息，也就是上面代码中的meta对象，然后将元信息保存到 _metadata.json 文件中

# vite 插件工作流 
和前文不同，这里注重resolvePlugins的实现上，Vite 所有的插件就是在这里被收集起来的。具体实现如下:
```js
export async function resolvePlugins( config: ResolvedConfig, prePlugins: Plugin[], normalPlugins: Plugin[], postPlugins: Plugin[]
): Promise<Plugin[]> 
{
    const isBuild = config.command === 'build' // 收集生产环境构建的插件，后文会介绍
    const buildPlugins = isBuild
    ? (await import('../build')).resolveBuildPlugins(config) : { pre: [], post: [] }
    return [
        // 1. 别名插件
        isBuild ? null : preAliasPlugin(),
        aliasPlugin({ entries: config.resolve.alias }), // 2. 用户自定义 pre 插件(带有`enforce: "pre"`属性) ...prePlugins,
        // 3. Vite 核心构建插件
        // 数量比较多，暂时省略代码
        // 4. 用户插件(不带有 `enforce` 属性)
        ...normalPlugins,
        // 5. Vite 生产环境插件 & 用户插件(带有 `enforce: "post"`属性) definePlugin(config),
        cssPostPlugin(config),
        ...buildPlugins.pre,
        ...postPlugins,
        ...buildPlugins.post,
        // 6. 一些开发阶段特有的插件
        ...(isBuild
        ? []
        : [clientInjectionsPlugin(config), importAnalysisPlugin(config)]) 
    ].filter(Boolean) as Plugin[]

```
vite 插件执行的具体顺序
1. 别名插件包括 vite:pre-alias 和 @rollup/plugin-alias ，用于路径别名替换。
2. 用户自定义 pre 插件，也就是带有 enforce: "pre" 属性的自定义插件。
3. Vite 核心构建插件，这部分插件为 Vite 的核心编译插件，数量比较多
4. 用户自定义的普通插件，即不带有 enforce 属性的自定义插件。 
5. Vite 生产环境插件 和用户插件中带有 enforce: "post" 属性的插件。
6. 一些开发阶段特有的插件，包括环境变量注入插件 clientInjectionsPlugin 和 import 语句分析及重写插件 importAnalysisPlugin 。

## vite 内置插件
1. 别名插件 
分别是 vite:pre-alias 和 @rollup/plugin-alias。前者主要是为了将 bare import 路径重定向到预构建依赖的路径，
后者则是实现了比较通用的路径别名(即 resolve.alias 配置)的功能
2. 核心构建插件
    - 当你在 Vite 配置文件中开启下面这个配置时:
        build: {
            polyfillModulePreload: true
        }
       Vite 会自动应用 modulePreloadPolyfillPlugin 插件，在产物中注入module preload的Polyfill 代码
    - 路径解析插件
        路径解析插件(即 vite:resolve )是Vite中比较核心的插件，几乎所有重要的 Vite特性都离不开这个插件的实现，诸如依赖预构建、HMR、SSR等等。同时它也是实现相当复杂的插件，一方面实现了 Node.js 官方的 resolve 算法.
    - 内联脚本加载插件
        对于 HTML中的内联脚本，Vite会通过 vite:html-inline-script-proxy 插件来进行加载。
    - css编译插件
        即名为 vite:css 的插件，主要实现下面这些功能:
        CSS 预处理器的编译 CSS Modules Postcss 编译
        通过 @import 记录依赖 ，便于 HMR
    - Esbuild 转译插件
        即名为 vite:esbuild 的插件，用来进行 .js 、 .ts 、 .jsx 和 tsx ，代替了传统的 Babel 或者 TSC 的功能，这也是Vite 开发阶段性能强悍的一个原因。插件中主要的逻辑是 transformWithEsbuild 函数，顾名思义，你可以通过这个函数进行代码转译。当然，Vite 本身也导出了这个函数，作为一种通用的 transform 能力。
    - 静态资源加载插件
      - vite:json vite:wasm vite:worker vite:asset
3. 生产环境特有插件
    - 全局变量替换插件
    - CSS 后处理插件
    - HTML 构建插件
    - Commonjs 转换插件
    - date-uri 插件
    - import-meta-url 支持插件
    - 压缩插件






