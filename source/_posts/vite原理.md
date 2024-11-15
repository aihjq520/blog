---
title: vite原理
date: 2022/7/5 01:20:00   
tags: 
- vite
- 构建工具
categories: 
- 前端
---


它解决的是开发的时候的效率问题，对于生产环境则是交给了 Rollup。


## vite是什么

`vite`有如下特点：

- 快速的冷启动: No Bundle + esbuild 预构建
- 即时的模块热更新: 基于ESM的HMR，同时利用浏览器缓存策略提升速度
- 真正的按需加载: 利用浏览器ESM支持，实现真正的按需加载

## Vite 原理

vite 主要通过 esbuild 预构建依赖和让浏览器接管部分打包程序两种手段解决了这两个问题，下面细讲这两大手段。

### esm是什么？

ESM是JavaScript提出的官方标准化模块系统，不同于之前的CJS，AMD，CMD等等，ESM提供了更原生以及更动态的模块加载方案，最重要的就是它是浏览器原生支持的，也就是说我们可以直接在浏览器中去执行import，动态引入我们需要的模块，而不是把所有模块打包在一起。

### 浏览器的表现

用vite打包的项目，在运行dev加载依赖时会发现script脚本标签都有`type="module"`这个属性

```JS
  <script type="module" src="/src/main.js"></script>
```

当浏览器解析资源时，会往当前域名发起一个GET请求main.js文件
```JS
// main.js
import { createApp } from 'vue'
import App from './App.vue'
createApp(App).mount('#app')
```

请求到了main.js文件，会检测到内部含有import引入的包，又会import 引用发起HTTP请求获取模块的内容文件，如App.vue、vue文件

Vite其核心原理是利用浏览器现在已经支持ES6的import,碰见import就会发送一个HTTP请求去加载文件，Vite启动一个 koa 服务器拦截这些请求，并在后端进行相应的处理将项目中使用的文件通过简单的分解与整合，然后再以ESM格式返回返回给浏览器。Vite整个过程中没有对文件进行打包编译，做到了真正的按需加载，所以其运行速度比原始的webpack开发编译速度快出许多！

## 预构建


### 为什么要预构建

1. 支持commonJS依赖

上面提到Vite是基于浏览器原生支持ESM的能力实现的，但要求用户的代码模块必须是ESM模块，因此必须将commonJs的文件提前处理，转化成 ESM 模块并缓存入 node_modules/.vite

2. 减少模块和请求数量

除此之外，我们常用的lodash工具库，里面有很多包通过单独的文件相互导入，而 lodash-es这种包会有几百个子模块，当代码中出现 import { debounce } from 'lodash-es' 会发出几百个 HTTP 请求，这些请求会造成网络堵塞，影响页面的加载。
 Vite 将有许多内部模块的 ESM 依赖关系转换为单个模块，以提高后续页面加载性能。

通过预构建 lodash-es 成为一个模块，也就只需要一个 HTTP 请求了！



### esbuild 预构建依赖

vite 将代码分为源码和依赖两部分并分别处理，所谓依赖便是应用使用的第三方包，一般存在于 node_modules 目录中，一个较大项目的依赖及其依赖的依赖，加起来可能达到上千个包，这些代码可能远比我们源码代码量要大，这些依赖通常是不会改变的（除非你要进行本地依赖调试），所以无论是 webpack 或者 vite 在启动时都会编译后将其缓存下来。区别的是，vite 会使用 esbuild 进行依赖编译和转换（commonjs 包转为 esm），而 webpack 则是使用 acorn 或者 tsc 进行编译，而 esbuild 是使用 Go 语言写的，其速度比使用 js 编写的 acorn 速度要快得多

### 源码依赖

Vite 是直接把转换后的 es module 的 JavaScript 代码，扔给支持 es module 的浏览器，让浏览器自己去加载依赖，也就是把压力丢给了浏览器，从而达到了项目启动速度快的效果。
源码模块的请求会根据 304 Not Modified 进行协商缓存，而依赖模块请求则会通过 Cache-Control: max-age=31536000,immutable 进行强缓存，因此一旦被缓存它们将不需要再次请求。

### 缓存

#### 文件系统缓存

Vite 会将预构建的依赖缓存到 node_modules/.vite。它根据几个源来决定是否需要重新运行预构建步骤:

- package.json 中的 dependencies 列表
- 包管理器的 lockfile，例如 package-lock.json, yarn.lock，或者 pnpm-lock.yaml
- 可能在 vite.config.js 相关字段中配置过的
  只有在上述其中一项发生更改时，才需要重新运行预构建。

如果出于某些原因，你想要强制 Vite 重新构建依赖，你可以用 --force 命令行选项启动开发服务器，或者手动删除 node_modules/.vite 目录

### 浏览器缓存

解析后的依赖请求会以 HTTP 头 max-age=31536000,immutable 强缓存，以提高在开发时的页面重载性能。一旦被缓存，这些请求将永远不会再到达开发服务器。如果安装了不同的版本（这反映在包管理器的 lockfile 中），则附加的版本 query 会自动使它们失效

### Rollup 打包

vite 针对的是现代浏览器，现代浏览器已经广泛支持了 ESM,我们能否像上古年代那样直接不经过打包，直接将代码部署到服务器？事实上，还是存在一些问题的：

首先，每个模块都会使用一个请求，可以想象一个应用会发出多少请求，这样即便使用 HTTP2 也会效率低下。
其次，现代应用为了性能，需要做 tree-shaking、懒加载和代码分割等优化，以减小应用的体积和更好地做浏览器缓存。
那么 vite 是否可以直接使用 esbuild 进行打包，保持开发和生产的统一？从长期看来是可以这样的，但是就目前而言，esbuild 对 css 和代码分割的支持不够友好，更多针对应用的构建能力还在持续开发中。因此，vite 选择了同样采用 ESM 格式的 rollup 来进行打包，并且 vite 的插件采用了 rollup 的 rollup-superset 接口，这使大部分的 rollup 的插件都能在 vite 上使用。
