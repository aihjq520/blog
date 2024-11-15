---
title: webpack常见优化手段.md
date: 2023/8/28 18:20:00   
tags: 
- webpack
- 构建工具
categories: 
- 前端
---

## webpack 如何优化前端性能



### 1.代码分割
- 提取第三方库「vendor」
```javascript
module.exports = {
    entry: {
        main: './src/index.js',
        vendor: ['react', 'react-dom'],
    },
}
```

- 依赖库分离「splitChunks」
```
optimization: {
  splitChunks: {
     chunks: "async", // 必须三选一： "initial" | "all"(推荐) | "async" (默认就是async)
     minSize: 30000, // 最小尺寸，30000
     minChunks: 1, // 最小 chunk ，默认1
     maxAsyncRequests: 5, // 最大异步请求数， 默认5
     maxInitialRequests : 3, // 最大初始化请求书，默认3
     automaticNameDelimiter: '~',// 打包分隔符
     name: function(){}, // 打包后的名称，此选项可接收 function
     cacheGroups:{ // 这里开始设置缓存的 chunks
         priority: 0, // 缓存组优先级
         vendor: { // key 为entry中定义的 入口名称
             chunks: "initial", // 必须三选一： "initial" | "all" | "async"(默认就是async)
             test: /react|lodash/, // 正则规则验证，如果符合就提取 chunk
             name: "vendor", // 要缓存的 分隔出来的 chunk 名称
             minSize: 30000,
             minChunks: 1,
             enforce: true,
             maxAsyncRequests: 5, // 最大异步请求数， 默认1
             maxInitialRequests : 3, // 最大初始化请求书，默认1
             reuseExistingChunk: true // 可设置是否重用该chunk
         }
     }
  }
 },

```

### 2.删除冗余代码 「Tree-Shaking」
是一种采用删除不需要的额外代码的方式优化代码体积的技术，tree-shaking会在打包过程中 静态分析 模块之间的导入导出，确定哪些模块导出值没有被使用饼打上标记，并将其删除，从而实现了打包产物的优化。

#### tree-shaking可以实现的基础
在之前CommonJs、AMD、CMD的模块化方案中，这种导入导出是动态的，难以预测的，因此在打包阶段，是无法分析哪些模块被使用，例如：

而ES 静态module方案下，模块之间的依赖关系是高度确定的静态的，与运行状态无关，可以进行可靠的静态分析，因此整个依赖树可以被静态地推导出解析语法树，可以做到在编译时候 分析ESM的模块，可以从代码字面量中推断出哪些模块没有被使用，这就是tree-shaking实现的必要条件。

### 3.按需加载
​ 当页面中一个文件过大并且还不一定用到的时候，我们希望在使用到的时才开始加载，这就是按需加载。要实现按需加载，

#### 组件库的按需加载 
在使用antd,element-ui的时候都会使用按需加载引入css，js文件。依赖一个插件babel-plugin-import

![](../images/9a4506dd70ea4a20b645e282782c31d4_tplv-k3u1fbpfcp-zoom-in-crop-mark_3024_0_0_0.awebp)

为何需要组件库按需加载?

组件库按需加载主要目的就是为了减少项目构建打包产物的大小，提高项目线上首屏渲染速度，减少白屏时间，减少流量消耗。


#### 组件库动态加载用法

### 方式一：手动加载

```javascript
import Button from 'vant/lib/button';
import 'vant/lib/button/style';

```

### 方式二：自动加载

安装 babel-plugin-import 插件

修改 babel 插件配置
```javascript

module.exports = {
  plugins: [
    ['import', {
      libraryName: 'vant',
      libraryDirectory: 'es',
      style: true
    }, 'vant']
  ]
};

```

#### 组件库按需加载的本质

#### 代码懒加载
 webpack 将源码中的 import、require 引入的文件编译之后再根据动态加载语法配置（通常以页面路由为基本单位）将较大的代码拆分并构建出较小的 chunk 包，应用在运行时执行到相应业务逻辑时才去加载执行对应 chunk 代码。 webpack 懒加载主要发生在下图的 JS 拆分出不同的 Chunk 这一过程中。
![](./a8df5aa127224fce9d997438039fb18b_tplv-k3u1fbpfcp-zoom-in-crop-mark_3024_0_0_0.awebp)




### 4.DllPlugin/DllReferencePlugin

https://www.cnblogs.com/skychx/p/webpack-dllplugin.html 


### gzipPlugin

服务端配置gzip压缩后可大大缩减资源大小。

### 5.HardSourceWebpackPlugin


### 6.swc-loader (基于rust编写)

受限于JS的语言本身效率的问题，近几年前端领域出现了不少工具被Rust重写，其中就包括编译JS/TS文件速度比Babel快不少的SWC，其所对标的工具就是Babel。SWC全称为Speed Web Compiler，其是基于Rust实现的工具，目前被很多前端知名项目（Next.js、Parcel和Deno）所使用。官方SWC的编译速度相对于Babel可提升近20倍。



