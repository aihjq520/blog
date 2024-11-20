---
title: 聊聊webpack SplitChunk
date: 2024/11/20 12:21:00   
tags: 
- webpack
- 构建工具
categories: 
- 前端
---



# 前言

这个webpack的内置插件在项目进行产物bundle优化的时候会经常用到，所以为了更加详细的了解它的配置，就搭建了demo项目，对不同的情况修改相对应的配置，并记录下来做出分析。

# splitChunk 是什么？

webpack 解析文件时，会生成一张依赖图，里面有许多 chunk 互相依赖 ，webpack 可以将这些 chunk 分割、整合成一个或多个 bundle。这一过程我们可以通过 SplitChunkPlugin 进行调整。

换言之就是对我们的asset(module)和进行拆分和整合。这里又可以引申出一个问题 `chunk`, `module`,`bundle`这些分别是什么概念和有什么区别？ 先看以下chunk的定义，

---

Chunk 则是输出产物的基本组织单位，在生成阶段 webpack 按规则将 entry 及其它 Module 插入 Chunk 中，之后再由 SplitChunksPlugin 插件根据优化规则与 ChunkGraph 对 Chunk 做一系列的变化、拆解、合并操作，重新组织成一批性能(可能)更高的 Chunks 。运行完毕之后 webpack 继续将 chunk 一一写入物理文件中，完成编译工作。

---

综上，Module 主要作用在 webpack 编译过程的前半段，解决原始资源“如何读”的问题；而 Chunk 对象则主要作用在编译的后半段，解决编译产物“如何写”的问题，两者合作搭建起 webpack 搭建主流程。


# chunk

如何需要先明白chunk 有两种形式：

`initial`： 是入口起点的 main chunk。此 chunk 包含为入口起点指定的所有模块及其依赖项。

`non-initia`l： 是可以延迟加载的块。可能会出现在使用 动态导入（import() 或者是 require()这种） 或者 SplitChunksPlugin 时。


webpack 会根据模块依赖图的内容组织分包 —— Chunk 对象，默认的分包规则有：

- 1. 同一个 entry 下触达到的模块组织成一个 chunk
- 2. 异步模块单独组织为一个 chunk
- 3. entry.runtime 单独组织成一个 chunk

默认规则集中在 compilation.seal 函数实现，seal 核心逻辑运行结束后会生成一系列的 Chunk、ChunkGroup、ChunkGraph 对象，后续如 SplitChunksPlugin 插件会在 Chunk 系列对象上做进一步的拆解、优化，最终反映到输出上才会表现出复杂的分包结果。


# 例子

用vue-cli快速搭建了一个demo项目，并且装上了 `vue-router`, `elment-plus`，加载了三个路由如下

![](../images/屏幕截图%202024-11-19%20144727.png)

其中我们在main.js 的vue实例上 `use` 了 `element-ui`，所以产出物会有。我们跑一下build,如下：

![](../images/屏幕截图%202024-11-19%20144626.png)

可以看到全部vue文件最后都被打包成了一个app.vue文件


我们改成动态路由的方式

![](../images/屏幕截图%202024-11-19%20151952.png)

然后再跑一下build

![](../images/屏幕截图%202024-11-19%20151916.png)

这样每个动态路由都被分割成了一个chunk。


# 问题

如果你稍微了解过 SplitChunk, 上面的结果不难理解。但有个问题是，如果我在异步路由即`ClinetA`里面引入了一个组件`componentA`, 然后 `RemoteB`也同时引入组件`componentA`， build之后这会发现这个`componentA`也产生了一个chunk，如下：


1. 异步路由`ClinetA`、`RemoteB`同时引入组件`componentA`


![](../images/屏幕截图%202024-11-20%20135350.png)



2. 只有异步路由`ClinetA`引入组件`componentA`


![](../images/屏幕截图%202024-11-20%20140347.png)



3. 同步路由`ClinetA`、`RemoteB`同时引入组件`componentA`

![](../images/屏幕截图%202024-11-20%20141655.png)

发现只有情况1才会产生新的chunk


弄懂了问题一的情况，也就可以解释为什么我在实际项目开发中，遇到了打开一个页面需要请求非常js文件的问题了。


![](../images/屏幕截图%202024-11-20%20140628.png)


我们的`componentA` 在异步路由 `ClientA`中被引入，所以webpack在打包的时候，同样的会当成异步引用。所我是猜测 `webpack` 会对引用次数2次及以上的异步文件 也会生出一个`chunk`，但是最终结果的话还是需要去看源码才能确定。


先说我的结论: **一个页面引入了许多同时也被其他页面引用的公用的组件,,导致这些公共组件在bundle的时候被拆分成chunk了**

如果我们实际开发过程中组件拆分得太细,会导致浏览器请求页面发出过多的http请求,造成阻塞,但是不拆分,代码也难以复用,所以我们要去衡量以下这个度,或者再去看看webpack有无其他配置项可以修改,例如 一个页面最多同时发起6个chunk, 或者使用http2也是个解决办法。
