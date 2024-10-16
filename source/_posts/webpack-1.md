---
title: webpack基础一（模块加载webpack_require）
date: 2023/5/25 19:30:00   
tags: 
- webpack
- 构建工具
categories: 
- 前端
---

# 1.  __webpack_moudles__和__webpack_require__
所有模块在 __webpack_modules__ 以键值对的形式等待加载，模块对象键为模块ID(路径) 值为模块的内容。 
模块内通过webpack封装的 __webpack_require__ 函数进行加载其他模块。

## __webpack_require__
```javascript
/**
   * 所有模块
   * 
   * 所有模块在 __webpack_modules__ 以键值对的形式等待加载，模块对象键为模块ID(路径) 值为模块的内容。 
   * 模块内通过webpack封装的 __webpack_require__ 函数进行加载其他模块
  */
  var __webpack_modules__ = ({
    "./src/index.js":
      (function (__unused_webpack_module, __unused_webpack_exports, __webpack_require__) {
        eval("const moduleA = __webpack_require__(/*! ./moduleA */ \"./src/moduleA.js\")\r\nconsole.log(\"this\", this);\r\nconsole.log(\"index.js,成功导入\" + moduleA.content);\n\n//# sourceURL=webpack://demo/./src/index.js?");
      }),

    "./src/moduleA.js":
      ((module, __unused_webpack_exports, __webpack_require__) => {
        eval("const moduleB = __webpack_require__( \"./src/moduleB.js\")\r\nconsole.log(\"moduleA模块,成功导入\" + moduleB.content);\r\nmodule.exports = {\r\n  content: \"moduleA模块\"\r\n}\n\n//# sourceURL=webpack://demo/./src/moduleA.js?");
      }),

    "./src/moduleB.js":
      ((module) => {
        eval("module.exports = {\r\n  content: \"moduleB模块\"\r\n}\n\n//# sourceURL=webpack://demo/./src/moduleB.js?");
      })
  });

 /**
   * webpack封装的加载函数  
   * 
   * 该函数根据模块ID加载模块，加载前先判断 是否模块缓存中是否有缓存，有的话直接使用缓存，
   * 没有则添加缓存并从所有模块(__webpack_modules__)中，加载这个模块
   */
  function __webpack_require__(moduleId) {
    // 加载前判断是否有可以使用的缓存
    var cachedModule = __webpack_module_cache__[moduleId];
    if (cachedModule !== undefined) {
      return cachedModule.exports;
    }
    // 创建一个新的空模块 并添加进缓存中
    var module = __webpack_module_cache__[moduleId] = {
      exports: {}
    };

    // 执行模块方法，执行过程中会加载模块中的代码 并获取模块的导出内容 
    __webpack_modules__[moduleId].call(module.exports, module, module.exports, __webpack_require__);

    // 返回模块最终导出的数据
    return module.exports;
  }
  
    /**
     * 这是bundle.js执行起点，通过 __webpack_require__ 导入配置入口文件的模块
     * 
     * 入口文件模块如果有依赖通过__webpack_require__ 继续导入,然后执行入口文件模块中的代码
     * "./src/index.js" 通过__webpack_exports__ 导入 "./src/moduleA.js"，"./src/moduleA.js" 通过__webpack_exports__ 导入"./src/moduleB.js"
     */
    var __webpack_exports__ = __webpack_require__("./src/index.js");
  /**
   * 执行流程
   * __webpack_require__ 会加载 __webpack_modules__中的某个模块，这个模块的执行方法会 递归调用 __webpack_require__ 加载下个模块知道所有模块加载完成
   */
})();
```



# 2. 执行顺序
2.1 获取源文件

// 中间可能会有loader

2.2 生成AST parse

2.3 语法转换 traverse
- require转换成__webpack_require__
- windows路径替换

2.4 生成代码 generator

2.5 重复上述2-4步骤，对收集到的依赖递归
在语法转换require转换成__webpack_require__的时候, 收集其他依赖。

2.6 最后生成__webpack_modules__， 保存了所有依赖的代码。是一个键值对结构。key是文件路径。



附上compiler的代码

```javascript
const path = require("path")
const fs = require("fs")
// 导入解析器
const parser = require("@babel/parser")
// 导入转换器   es6导出需要.defult
const traverse = require("@babel/traverse").default

// 导入生成器
const generator = require("@babel/generator").default

// 导入ejs
const ejs = require("ejs")

// 导入 tabable
const { SyncHook } = require("tapable")
class Compiler {
  constructor(config) {
    this.config = config
    this.entry = config.entry
    // process.cwd可以获取node执行的文件绝对路径
    // 获取被打包项目的文件路径
    this.root = process.cwd()

    // 存放打包后的所有模块
    this.modules = {}

    // 获取所有loader
    this.rules = config.module && config.module.rules

    // 1、声明钩子  
    this.hooks = {
      compile: new SyncHook(),
      afterCompile: new SyncHook(),
      emit: new SyncHook(),
      afterEmit: new SyncHook(['modules']),
      done: new SyncHook()
    }

    // 2、获取所有插件对象，执行插件apply方法
    if (Array.isArray(this.config.plugins)) {
      this.config.plugins.forEach(e => {
        // 传入compiler 实例，插件可以在apply方法中注册钩子
        e.apply(this)
      })
    }
  }
  // 根据传入的文件路径，解析文件模块
  depAnalyse(modulePath) {
    let code = this.getSource(modulePath)
    // 使用loader处理源码
    code = this.useLoader(code, modulePath)

    // 将代码解析为ast抽象语法树
    const ast = parser.parse(code)

    // 当前模块依赖数组,存放当前模块所以依赖的路径  
    let dependencies = []

    /**
     * traverse用来转换语法，它接收两个参数 
     *   - ast:转换前的抽象语法树节点树  
     *   - options: 配置对象，里面包含各种钩子回调，traverse遍历语法树节点当节点满足某个钩子条件时  该钩子会被触发
     *     - CallExpression：
     */
    traverse(ast, {
      // 当某个抽象语法树节点类型为 CallExpression（表达式调用），会触发该钩子
      CallExpression(p) {
        if (p.node.callee.name === 'require') {
          // 修改require
          p.node.callee.name = "__webpack_require__"

          // 修改当前模块 依赖模块的路径 使用node访问资源必须是 "./src/XX" 的形式
          let oldValue = p.node.arguments[0].value
          // 将"./xxx" 路径 改为 "./src/xxx"
          oldValue = "./" + path.join("src", oldValue)

          // 避免window的路径出现 "\"
          p.node.arguments[0].value = oldValue.replace(/\\+/g, "/")

          // 每解析require，就将依赖模块的路径放入 dependencies 中
          dependencies.push(p.node.arguments[0].value)
        }
      },
    })

    const sourceCode = generator(ast).code

    // 模块得ID处理为相对路径  
    let modulePathRelative = "./" + path.relative(this.root, modulePath)
    // 将路径中 "\" 替换为 "/"
    modulePathRelative = modulePathRelative.replace(/\\+/g, "/")
    // 当前模块解析完毕 将其添加进modules
    this.modules[modulePathRelative] = sourceCode


    // 如果当前模块 有其他依赖的模块就 递归调用 depAnalyse 继续向下解析代码 直到没有任何依赖为止
    dependencies.forEach(depPath => {
      // 传入模块的绝对路径
      this.depAnalyse(path.resolve(this.root, depPath))
    })
  }
  // 传入文件路径，读取文件
  getSource(path) {
    // 以“utf-8”编码格式 读取文件并返回
    return fs.readFileSync(path, "utf-8")
  }
  // 执行解析器
  start() {
    // 开始解析前，执行 compile 钩子
    this.hooks.compile.call()
    // 传入入口文件绝对路径 开始解析依赖
    // 注意：此处路径不能使用 __dirname，__dirname代表 工具库"my-webpack"根目录的绝对路径  而不是要被打包项目的根目录路径
    this.depAnalyse(path.resolve(this.root, this.entry))
    // 分析结束后，执行 afterCompile 钩子 
    this.hooks.afterCompile.call()
    // 资源发射前，执行 emit 钩子
    this.hooks.emit.call()
    this.emitFile()
    // 资源发射完毕，执行 afterEmit 钩子
    this.hooks.afterEmit.call()
    // 解析结束，执行 done 钩子
    this.hooks.done.call()
  }
  // 生成文件
  emitFile() {
    // 读取代码渲染依据的模板
    const template = this.getSource(path.resolve(__dirname, "../template/output.ejs"))
    // 传入渲染模板、模板中用到的变量
    let result = ejs.render(template, {
      entry: this.entry,
      modules: this.modules
    })
    // 获取输出路径 
    let outputPath = path.join(this.config.output.path, this.config.output.filename)
    // 生成bundle文件
    fs.writeFileSync(outputPath, result)
  }
  // 使用loader
  useLoader(code, modulePath) {
    // 未配置rules 直接返回源码
    if (!this.rules) return code
    // 获取源码后 输出给loader ,倒序遍历loader  
    for (let index = this.rules.length - 1; index >= 0; index--) {
      const { test, use } = this.rules[index]
      // 如果当前模块满足 loader正则匹配  
      if (test.test(modulePath)) {
        /**
         * 如果单个匹配规则，多个loader，use字段为数组。单个loader，use字段为字符串或对象
         */
        if (use instanceof Array) {
          // 倒序遍历loader 传入源码进行处理
          for (let i = use.length - 1; i >= 0; i--) {
            let loader = use[i];
            /**
             * loader 为字符串或对象
             *   字符串形式：
             *   use:["loader1","loader2"]
             *   对象形式：
             *   use:[
             *     {
             *        loader:"loader1",
             *        options:....
             *     }
             *   ]
             */
            let loaderPath = typeof loader === 'string' ? loader : loader.loader
            // 获取loader的绝对路径
            loaderPath = path.resolve(this.root, loaderPath)
            // loader 上下文
            const options = loader.options
            const loaderContext = {
              getOptions() {
                return options
              }
            }
            // 导入loader
            loader = require(loaderPath)
            // 传入上下文 执行loader 处理源码
            code = loader.call(loaderContext, code)
          }
        } else {
          let loaderPath = typeof loader === 'string' ? use : use.loader
          // 获取loader的绝对路径
          loaderPath = path.resolve(this.root, loaderPath)
          // loader 上下文
          const loaderContext = {
            getOptions() {
              return use.options
            }
          }
          // 导入loader
          let loader = require(loaderPath)
          // 传入上下文 执行loader 处理源码
          code = loader.call(loaderContext, code)
        }
      }
    }
    return code
  }
}
module.exports = Compiler
```
