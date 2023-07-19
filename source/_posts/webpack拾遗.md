---
title: webpack拾遗
date: 2022-05-10 21:17:14
tags: [webpack]
categories: [webpack]
---
记录下有关webpack的一些基础知识以及工作中使用到的。只是一个备忘录。

# loader和plugin 工作中自定义loader

- loader 用于对模块的"源代码"进行转换
- plugin赋予其各种灵活的功能，例如打包优化、资源管理、环境变量注入等，它们会运行在 webpack 的不同阶段（钩子 / 生命周期），贯穿了webpack整个编译

工作中使用到的loader， 因为当时前端在打包输出需要有两份格式结果。一份用于正常的网络版本，一份用于客户端的离线版本。人为的打包输出太麻烦，这是就需要脚本协助。webpack loader的编写形式如下：
```js
// 导出一个函数，source为webpack传递给loader的文件源内容
module.exports = function(source) {
    const content = doSomeThing2JsString(source);
    
    // 如果 loader 配置了 options 对象，那么this.query将指向 options
    const options = this.query;
    
    // 可以用作解析其他模块路径的上下文
    console.log('this.context');
    
    /*
     * this.callback 参数：
     * error：Error | null，当 loader 出错时向外抛出一个 error
     * content：String | Buffer，经过 loader 编译后需要导出的内容
     * sourceMap：为方便调试生成的编译后内容的 source map
     * ast：本次编译生成的 AST 静态语法树，之后执行的 loader 可以直接使用这个 AST，进而省去重复生成 AST 的过程
     */
    this.callback(null, content); // 异步
    return content; // 同步
}
```

```js
// 实际项目时使用 简写
module.exports = function(source) {
    if(process.env.BUILD_MODE === 'offline') {
        // replace 相关的cdn
    }else {
        //去除无关的符号
    }
    this.callback(null, content); // 异步
}
```


# webpack热更新
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230720001858.png)

webpack中配置
```js
const webpack = require('webpack')
module.exports = {
  // ...
  devServer: {
    // 开启 HMR 特性
    hot: true
    // hotOnly: true
  }
}
```
当某一个文件或者模块发生变化时，webpack监听到文件变化对文件重新编译打包，编译生成唯一的hash值，这个hash值用来作为下一次热更新的标识

根据变化的内容生成两个补丁文件：manifest（包含了 hash 和 chundId，用来说明变化的内容）和chunk.js 模块

由于socket服务器在HMR Runtime 和 HMR Server之间建立 websocket链接，当文件发生改动的时候，服务端会向浏览器推送一条消息，消息包含文件改动后生成的hash值，如下图的h属性，作为下一次热更细的标识
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230720002205.png)
在浏览器接受到这条消息之前，浏览器已经在上一次socket 消息中已经记住了此时的hash 标识，这时候我们会创建一个 ajax 去服务端请求获取到变化内容的 manifest 文件

mainfest文件包含重新build生成的hash值，以及变化的模块，对应上图的c属性

浏览器根据 manifest 文件获取模块变化的内容，从而触发render流程，实现局部模块更新
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230720002306.png)

# webpack工作流程
![](https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230720001614.png)

1. 根据config文件生成option
2. 从entry-point开始递归分析依赖，对每个以来模块进行build
3. 将通过loader生成的module编译成ast树
4. 遍历ast树，进行build 输出的dist目录