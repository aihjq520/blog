---
title: webpack基础三（plguin）
date: 2023/8/23 19:30:00   
tags: 
- webpack
- 构建工具
categories: 
- 前端
---

webpack 在整个编译周期中会触发很多不同的事件，plugin 可以监听这些事件，并且可以调用 webpack 的 API 对输出资源进行处理。

这是它和 loader 的不同之处，loader 一般只能对源文件代码进行转换，而 plugin 可以做得更多。plugin 在整个编译周期中都可以被调用，只要监听事件。插件的范围包括，从打包优化和压缩，一直到重新定义环境中的变量。插件接口功能极其强大，可以用来处理各种各样的任务。

在插件开发中最重要的两个资源就是 compiler 和 compilation 对象。理解它们的角色是扩展 webpack 引擎重要的第一步。
> compiler 对象代表了完整的 webpack 环境配置。这个对象在启动 webpack 时被一次性建立，并配置好所有可操作的设置，包括 options，loader 和 plugin。当在 webpack 环境中应用一个插件时，插件将收到此 compiler 对象的引用。可以使用它来访问 webpack 的主环境。

> compilation 对象代表了一次资源版本构建。当运行 webpack 开发环境中间件时，每当检测到一个文件变化，就会创建一个新的 compilation，从而生成一组新的编译资源。一个 compilation 对象表现了当前的模块资源、编译生成资源、变化的文件、以及被跟踪依赖的状态信息。compilation 对象也提供了很多关键时机的回调，以供插件做自定义处理时选择使用

我们看一下官网的定义，webpack 插件由以下部分组成：


<ol>
<li>一个 JavaScript 命名函数。</li>
<li>在插件函数的 prototype 上定义一个 apply 方法。</li>
<li>指定一个绑定到 webpack 自身的事件钩子。</li>
<li>处理 webpack 内部实例的特定数据。</li>
<li>功能完成后调用 webpack 提供的回调。</li>
</ol>

简单的说，一个具有 apply 方法的函数就是一个插件，并且它要监听 webpack 的某个事件。下面来看一个简单的示例：

```javascript

function Plugin(){}

Plugin.prototype.apply = function(compiler){
    compiler.plugin('emit', function(compilation, callback){
        callback()
    })
}

module.exports = Plugin

```

写完插件后要怎么调用呢？

先在 webpack 配置文件中引入插件，然后在 plugins 选项中配置：

```javascript 
const Plugin = require('./src/plugin')

module.exports = {
	...
    plugins: [
        new Plugin()
    ]
}

```

### html-webpack-plugun

作用: 当使用 webpack打包时，创建一个 html 文件，并把 webpack 打包后的静态文件自动插入到这个 html 文件当中。

#### 使用默认配置
```javascript
const HtmlWebpackPlugin = require('html-webpack-plugin')

module.exports = {
  entry: 'index.js',
  output: {
    path: __dirname + '/dist',
    filename: 'index_bundle.js'
  },
  plugins: [
    new HtmlWebpackPlugin()
  ]
}
```

#### 使用自定义模版生成 html 文件
如果默认的 html 模版不能满足业务需求， 比如需提前写一些 css 'js' 资源的引用， 最简单的方式就是新建一个模版文件， 并使用 template 属性指定模版文件的路径，html-webpack-plugin 插件将会自动向这个模版文件中注入打包后的 js 'css' 文件资源。
```javascript
const HtmlWebpackPlugin = require('html-webpack-plugin')

module.exports = {
  entry: 'index.js',
  output: {
    path: __dirname + '/dist',
    filename: 'index_bundle.js'
  },
  plugins: [
  new HtmlWebpackPlugin({
    title: 'My App', 
    template: 'public/index.html',
    minify:{
                removeComments: true, //去除HTML注释
                collapseWhitespace: true, //去掉空格
                minifyJS: true, // 压缩html里的js（使用uglify-js进行的压缩）
                minifyCSS: true, // 压缩html里的css（使用clean-css进行的压缩）
            }
  })
]
}
```

### mini-css-extract-plugin
作用：该插件的主要是为了抽离 css 样式,防止将样式打包在 js 中文件过大和因为文件大网络请求超时的情况。


### defineplugin
作用：DefinePlugin 允许在 编译时 将你代码中的变量替换为其他值或表达式。这在需要根据开发模式与生产模式进行不同的操作时，非常有用。

```javascript
new webpack.DefinePlugin({
  PRODUCTION: JSON.stringify(true),
  VERSION: JSON.stringify('5fa3b9'),
  BROWSER_SUPPORTS_HTML5: true,
  TWO: '1+1',
  'typeof window': JSON.stringify('object'),
  'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV),
});
```

在项目中具体用到的地方？ 没有dev和build不一样的地方， 但会用到注入变量。

### DllPlugin
此插件用于在单独的 webpack 配置中创建一个 dll-only-bundle。 此插件会生成一个名为 manifest.json 的文件，这个文件是用于让 DllReferencePlugin 能够映射到相应的依赖上。

https://www.cnblogs.com/skychx/p/webpack-dllplugin.html (webpack4不再适用)

原理：

1. 产出物[name].dll.js 最终导出的是一个webpack__require函数
2. 在bundle.js中在请求dll打包后的产出物

```javascript

var __webpack_modules__ = {
    // 模块B
    "./node_modules/isarray/index.js": (
      module,
      __unused_webpack_exports,
      __webpack_require__
    ) => {

      // 加载模块<dll-reference _dll_utils>
      module.exports = __webpack_require__("dll-reference _dll_utils")(
        "./node_modules/isarray/index.js"
      );
    },

    "dll-reference _dll_utils": (module) => {
      "use strict";
      module.exports = _dll_utils;
    },
  };

```

所以当页面中去使用isArray时，去使用dll_utils导出的webpack__require函数去引入isArray


https://juejin.cn/post/7043653757053173797



难点1： 如何将dll的产物附加到index.prod.html上?





### ProviderPlugin

假设你有一个 main.js 文件，代码如下：

```javascript
console.log(window.Quill);
```


使用 ProvidePlugin 插件后，Webpack 会将其转换为：


```javascript
var window = window || {};
window.Quill = require('quill');
console.log(window.Quill);
```

配置项

```javascript
const webpack = require('webpack');
          
module.exports = {
  chainWebpack: config => {
    // 使用 ProvidePlugin 插件
    config.plugin("provide").use(webpack.ProvidePlugin, [
      {
        "window.Quill": "quill",
      },
    ]);
  }
};
```

- config.plugin("provide")：在 Webpack 配置中定义一个名为 provide 的插件。
- .use(webpack.ProvidePlugin, [...])：使用 webpack.ProvidePlugin 插件，并传递配置选项。
- [{ "window.Quill": "quill" }]：配置选项表示，当在代码中遇到 window.Quill 时，自动将其替换为 require('quill')。


需要特别注意在ts环境下， 用providerplugin 一般都会报 “cannot find name xxx” 这种错误
需要在全局声明文件中定义

```typescript
declare module xxx {

}
```

https://segmentfault.com/a/1190000022509809