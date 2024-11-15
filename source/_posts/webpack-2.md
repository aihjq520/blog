---
title: webpack基础二（loader）
date: 2023/7/7 19:30:00   
tags: 
- webpack
- 构建工具
categories: 
- 前端
---

## 原理

官网上的定义：

loader 是一个转换器，用于对源代码进行转换。 

例如 babel-loader 可以将 ES6 代码转换为 ES5 代码；sass-loader 将 sass 代码转换为 css 代码。

loader 的实现
```javascript
module.exports = function (source) {
    return source.replace(/var/g, 'const')
}

```


```javascript
const path = require('path')

module.exports = {
    mode: 'development',
    entry: {
        main: './src/index.js'
    },
    output: {
        filename: 'bundle.js',
        path: path.resolve(__dirname, 'dist')
    },
    module: {
        rules: [
            {
                test: /\.js$/,
                use: [
                    {
                        loader: path.resolve('./src/loader1.js'),
                    },
                ]
            }
        ]
    },
}
```


下面我们来实现一个异步 loader：
```
module.exports = function (source) {
    const callback = this.async()

    // 由于有 3 秒延迟，所以打包时需要 3+ 秒的时间
    setTimeout(() => {
        callback(null, `${source.replace(/;/g, '')}`)
    }, 3000)
}
```

异步 loader 需要调用 webpack 的 async() 生成一个 callback，它的第一个参数是 error，这里可设为 null，第二个参数就是处理后的源码。当你异步处理完源码后，调用 callback 即可。


### 处理css

- css-loader 是css-loader只是帮我们解析了css文件里面的css代码，
默认webpack是只解析js代码的，所以想要应用样式我们要把解析完的css代码拿出来加入到
style标签中。 
- style-loader 就是帮我们直接将css-loader解析后的内容挂载到html页面当中，我们来看一下
- postcss-loader 处理css属性浏览器兼容性
- scss-loader 处理scss文件

```javascript

module:{
    rules:[
        {
            test:/\.css$/,
            use:[
                'style-loader'
                'css-loader',
            ]
        }
    ]
}
```

### 处理图片
如果页面图片较多，发很多 http 请求，会降低页面性能。这个问题可以通过 url-loader 解决。url-loader 会将引入的图片以 base64 编码并打包到文件中，最终只需要引入这个dataURL 就能访问图片了。当然，如果图片较大，编码会消耗性能。因此 url-loader 提供了一个 limit 参数，小于 limit 字节的文件会被转为 base64，大于 limit 的会使用 file-loader 的参数进行命名，并把图片 copy 到指定文件夹内。

- file-loader
- url-loader 
只处理两种图片引入情况
React 组件内部直接引入
```javascript
import red_png from "./red.png";
<img src = {red_png} />
```

css 背景图引入
```css
background-image: url("./images/blue.png");
```

### 处理ts的方式
在webpack中，编译ts文件有两种方式。

- 1.使用ts-loader编译。

```javascript

modules:{
    rules:[
        {
            test:/\.ts^/
            use:[ts-loader],

        }
    ]
}

```

配置ts.config.json
ts.config.json文件是必须要创建的，用来配置ts-loader的参数。
内容可以根据自己项目具体要求来进行配置。如果对ts的配置文件不熟悉的话可以看看笔者前面写的TypeScript学习之配置文件。
这里我们简单配置下


- 2.使用babel-loader编译

从7.X开始，babel开始支持ts的编译，如果想让babel编译ts，需要指定babel的preset。(cra是用babel编译ts的)

```javascript
npm i babel-loader @babel/preset-env @babel/preset-typescript core-js@3 -D
```

先使用@babel/preset-typescript来识别ts，然后使用@babel/preset-env来编译js和按需引入polyfill

```javascript

{
  "presets": [
    [
      "@babel/preset-env",
      {
        "useBuiltIns": "usage",
        "corejs": "3"
      }
    ],
    "@babel/preset-typescript"
  ]
}

```

所以使用babel-loader来处理ts的好处就是能动态按需引入polyfill。这个在ts-loader是不支持的，如果需要引入polyfill，需要在入口文件全量引入polyfill。

#### 生成类型申明文件

使用babel-loader的时候并没有使用到tsc，也不会读取ts.config.json文件配置，所以使用babel-loader是不能生成类型申明文件。


####  两种方式的区别
### polyfill
ts-loader不支持polyfill的按需引入，如果需要polyfill需要在入口全量引入。
babel-loader是可以按需引入polyfill。并且还可以配置@babel/plugin-transform-runtime插件来抽离公共辅助函数。
### 类型申明文件
ts-loader支持生成类型申明文件，只需要在ts.config.json中配置即可。而babel-loader不支持。

### 错误检测
ts-loader支持错误检测，如果语法有问题会编译不成功。babel-loader是不支持，需要单独利用tsc单独配置。
因为 tsc 的类型检查是需要拿到整个工程的类型信息，需要做类型的引入、多个文件的 namespace、enum、interface 等的合并，而 babel 是单个文件编译的，不会解析其他文件的信息。所以做不到和 tsc 一样的类型检查。


### 编译速度
ts-loader编译速度慢，因为它有类型检查这一步。babel-loader编译速度快。

### 构建体积
babel-loader打包后的体积更小。因为它支持按需polyfill和@babel/plugin-transform-runtime插件的配置。
其次babel-loader可以通过.browserslistrc或target精确配置需要适配的浏览器。而ts-loader只能配置target，其值有'ES3' (default), 'ES5', 'ES6'/'ES2015', 'ES2016', 'ES2017', or 'ESNEXT'，相对没那么精确。所以对打包体积也会有影响。

