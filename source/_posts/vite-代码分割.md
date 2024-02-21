---
title: 代码分割
date: 2023-12-07 22:12:09
tags: [vite]
categories: [vite]
---
Vite 作为一个完整的构建工具，本身实现了一套 HMR 系统，值得注意的是，这套 HMR 系统基于原生的 ESM 模块规范来实现，在文件发生改变时 Vite 会侦测到相应 ES 模块的 变化，从而触发相应的 API，实现局部的更新