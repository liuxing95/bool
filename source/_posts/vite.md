---
title: vite介绍
date: 2021-04-13 11:52:21
categories: 构建工具
tags: vite
---

# Vite原理分析

## Vite是什么

```
Vite 基于浏览器原声 ES Module 的开发服务器 从某种意义上来说用来取代 webpack-dev-server。

利用浏览器去解析模块 在服务端按需编译返回， 跳过了打包的概念 服务器随起随用， 搞定了热更新。

```
`Vite` 是基于开发环境使用的构建工具 在生产环境下是 基于 `Rollup` 打包

### Vite的特点
1. 冷启动速度快
2. 即时热模块更新（热更新）
3. 真正的按需编译

Vite要求项目完全由 ES module 模块组成， common.js模块不能直接在 Vite 上使用

### ES Modules
ES Modules 是浏览器支持的一种模块化方案，允许在代码中实现模块化


## Vite实现

### 请求拦截原理
Vite的基本实现原理 就是启动一个 http 服务器 拦截浏览器请求 ES Module 的请求 。 通过path 找到目录下对应的文件做一定的处理最终以 ES Modules 格式返回给客户端


<img src="https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3157aa930e1f44eaa501412b9f4ea576~tplv-k3u1fbpfcp-zoom-1.image" width="500">

#### node_modules 模块的处理
在之前 我们写代码时 直接饮用 node_modules 的模块 是 采用下面的这种格式

```js
import React from 'react'
```
如 Webpack & gulp 等打包工具会帮我们找到模块的路径。但浏览器只能通过相对路径去寻找。

为了解决这个问题，Vite对其做了一些特殊处理。




