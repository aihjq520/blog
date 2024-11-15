---
title: webpack5打包产物分析
date: 2023/10/20 18:20:00   
tags: 
- webpack
- 构建工具
categories: 
- 前端
---


首先会打出一个类似bootstrap功能的runtime文件，**其实就是一个 IIFE**，会初始化__webpack_require__函数以及给他的原型链添加一些r,n,o等名字的函数，包括
- webpack/runtime/hasOwnProperty shorthand  (__webpack_require__.o)
- define __esModule on exports (__webpack_require__.r)

最重要的其实是__webpack_require__函数，看下他的实现

```javascript
function __webpack_require__(moduleId) {
/******/ 		// Check if module is in cache
/******/ 		var cachedModule = __webpack_module_cache__[moduleId];
/******/ 		if (cachedModule !== undefined) {
/******/ 			return cachedModule.exports;
/******/ 		}
/******/ 		// Create a new module (and put it into the cache)
/******/ 		var module = __webpack_module_cache__[moduleId] = {
/******/ 			// no module.id needed
/******/ 			// no module.loaded needed
/******/ 			exports: {}
/******/ 		};
/******/ 	
/******/ 		// Execute the module function
/******/ 		__webpack_modules__[moduleId](module, module.exports, __webpack_require__);
/******/ 	
/******/ 		// Return the exports of the module
/******/ 		return module.exports;
/******/ 	}
```

初始化了一个全局缓存模块对象，id即文件路径，当作key值。然后会为这个文件路径创建一个module对象，这个对象的exports key的value是这个文件的导出对象。这个exports的赋值是在
__webpac__modules[moduleId]这一行，这句的代码是执行了这个moduleId文件的内容。

```javascript
var __webpack_modules__ = ({

/***/ "./src/sayHello.js":
/*!*************************!*\
  !*** ./src/sayHello.js ***!
  \*************************/
/***/ ((__unused_webpack_module, __webpack_exports__, __webpack_require__) => {

__webpack_require__.r(__webpack_exports__);
/* harmony export */ __webpack_require__.d(__webpack_exports__, {
/* harmony export */   "default": () => (__WEBPACK_DEFAULT_EXPORT__)
/* harmony export */ });
function sayHello(name) {
  return `Hello ${name}`;
}

/* harmony default export */ const __WEBPACK_DEFAULT_EXPORT__ = (sayHello);

/***/ })    

/******/ 	});
```

可以看到打包后的文件与源码文件的有一个差异是ESM的export 和 import被webpack转换了。

## webpack 是如何支持 ESM 的

ESM的export 和 import被webpack转换了。
如果文件写的是export default 那么源码转换时是会往module对添加一个“default”: fn， 如果是类似export {key}，那么就会往module对象添加一个"key": fn。
如果文件是import， 那么会根据是import a from './xx.js' 转换成 _sayHello__WEBPACK_IMPORTED_MODULE_0__["default"], 如果是写的 import {dn} from './b.js' 则会被转换成
_export__WEBPACK_IMPORTED_MODULE_1__.dn


## 动态导入 

主要思想是使用 JSONP 动态加载，主要逻辑在三个函数
- __webpack_require__.e
- __webpack_require__.f
- __webpack_require__.l

当我们在源码写import('./another.js')，经过webpack编译后会变成
```javascript
__webpack_require__.e(/*! import() */ "src_another_js").then(__webpack_require__.bind(__webpack_require__, /*! ./another.js */ "./src/another.js")).then(res => console.log(res))
```

__webpack_require_e主要的实如下：
```javascript
(() => {
/******/ 		__webpack_require__.f = {};
/******/ 		// This file contains only the entry chunk.
/******/ 		// The chunk loading function for additional chunks
/******/ 		__webpack_require__.e = (chunkId) => {
/******/ 			return Promise.all(Object.keys(__webpack_require__.f).reduce((promises, key) => {
/******/ 				__webpack_require__.f[key](chunkId, promises);
/******/ 				return promises;
/******/ 			}, []));
/******/ 		};
/******/ 	})();
```
这段代码的意思是用promise.all去加载__webpack_require__.f的所有key值promise，那看下__webpack_require__.f的key值,添加了j函数

```javascript
var installedChunks = {
/******/ 			"main": 0
/******/ 		};
/******/ 		
/******/ 		__webpack_require__.f.j = (chunkId, promises) => {
/******/ 				// JSONP chunk loading for javascript
/******/ 				var installedChunkData = __webpack_require__.o(installedChunks, chunkId) ? installedChunks[chunkId] : undefined;
/******/ 				if(installedChunkData !== 0) { // 0 means "already installed".
/******/ 		
/******/ 					// a Promise means "currently loading".
/******/ 					if(installedChunkData) {
/******/ 						promises.push(installedChunkData[2]);
/******/ 					} else {
/******/ 						if(true) { // all chunks have JS
/******/ 							// setup Promise in chunk cache
/******/ 							var promise = new Promise((resolve, reject) => (installedChunkData = installedChunks[chunkId] = [resolve, reject]));
/******/ 							promises.push(installedChunkData[2] = promise);
/******/ 		
/******/ 							// start chunk loading
/******/ 							var url = __webpack_require__.p + __webpack_require__.u(chunkId);
/******/ 							// create error before stack unwound to get useful stacktrace later
/******/ 							var error = new Error();
/******/ 							var loadingEnded = (event) => {
/******/ 								if(__webpack_require__.o(installedChunks, chunkId)) {
/******/ 									installedChunkData = installedChunks[chunkId];
/******/ 									if(installedChunkData !== 0) installedChunks[chunkId] = undefined;
/******/ 									if(installedChunkData) {
/******/ 										var errorType = event && (event.type === 'load' ? 'missing' : event.type);
/******/ 										var realSrc = event && event.target && event.target.src;
/******/ 										error.message = 'Loading chunk ' + chunkId + ' failed.\n(' + errorType + ': ' + realSrc + ')';
/******/ 										error.name = 'ChunkLoadError';
/******/ 										error.type = errorType;
/******/ 										error.request = realSrc;
/******/ 										installedChunkData[1](error);
/******/ 									}
/******/ 								}
/******/ 							};
/******/ 							__webpack_require__.l(url, loadingEnded, "chunk-" + chunkId, chunkId);
/******/ 						}
/******/ 					}
/******/ 				}
/******/ 		};

```

设置 chunk 加载的三种状态并缓存在installedChunks中，防止chunk重复加载。这些状态的改变会在 webpackJsonpCallback 中提到

installedChunkData = installedChunks[chunkId] = [resolve, reject];

installedChunks[chunkId]为0，代表该 chunk 已经加载完毕
installedChunks[chunkId]为undefined，代表该 chunk 加载失败、加载超时、从未加载过
installedChunks[chunkId]为Promise对象，代表该 chunk 正在加载

最后使用__webpack_require__.l加载对应chunkID的js文件

```javascript

__webpack_require__.l = (url, done, key, chunkId) => {
/******/ 			if(inProgress[url]) { inProgress[url].push(done); return; }
/******/ 			var script, needAttach;
/******/ 			if(key !== undefined) {
/******/ 				var scripts = document.getElementsByTagName("script");
/******/ 				for(var i = 0; i < scripts.length; i++) {
/******/ 					var s = scripts[i];
/******/ 					if(s.getAttribute("src") == url || s.getAttribute("data-webpack") == dataWebpackPrefix + key) { script = s; break; }
/******/ 				}
/******/ 			}
/******/ 			if(!script) {
/******/ 				needAttach = true;
/******/ 				script = document.createElement('script');
/******/ 		
/******/ 				script.charset = 'utf-8';
/******/ 				script.timeout = 120;
/******/ 				if (__webpack_require__.nc) {
/******/ 					script.setAttribute("nonce", __webpack_require__.nc);
/******/ 				}
/******/ 				script.setAttribute("data-webpack", dataWebpackPrefix + key);
/******/ 		
/******/ 				script.src = url;
/******/ 			}
/******/ 			inProgress[url] = [done];
/******/ 			var onScriptComplete = (prev, event) => {
/******/ 				// avoid mem leaks in IE.
/******/ 				script.onerror = script.onload = null;
/******/ 				clearTimeout(timeout);
/******/ 				var doneFns = inProgress[url];
/******/ 				delete inProgress[url];
/******/ 				script.parentNode && script.parentNode.removeChild(script);
/******/ 				doneFns && doneFns.forEach((fn) => (fn(event)));
/******/ 				if(prev) return prev(event);
/******/ 			}
/******/ 			var timeout = setTimeout(onScriptComplete.bind(null, undefined, { type: 'timeout', target: script }), 120000);
/******/ 			script.onerror = onScriptComplete.bind(null, script.onerror);
/******/ 			script.onload = onScriptComplete.bind(null, script.onload);
/******/ 			needAttach && document.head.appendChild(script);
/******/ 		};


```

## 异步 Chunk

异步chunk的js代码如下：
```javascript
"use strict";
(self["webpackChunklearn_webpack_output"] = self["webpackChunklearn_webpack_output"] || []).push([["src_another_js"],{

/***/ "./src/another.js":
/*!************************!*\
  !*** ./src/another.js ***!
  \************************/
/***/ ((__unused_webpack_module, __webpack_exports__, __webpack_require__) => {

__webpack_require__.r(__webpack_exports__);
/* harmony export */ __webpack_require__.d(__webpack_exports__, {
/* harmony export */   Another: () => (/* binding */ Another)
/* harmony export */ });
function Another() {
  return 'Hi, I am Another Module';
}

/***/ })

}]);
```
在加载异步chunk之后，会执行push操作，这个push其实是经过改写的, 其实是调用的webpackJsonpCallback:

```javascript
// install a JSONP callback for chunk loading
/******/ 		var webpackJsonpCallback = (parentChunkLoadingFunction, data) => {
/******/ 			var [chunkIds, moreModules, runtime] = data;
/******/ 			// add "moreModules" to the modules object,
/******/ 			// then flag all "chunkIds" as loaded and fire callback
/******/ 			var moduleId, chunkId, i = 0;
/******/ 			if(chunkIds.some((id) => (installedChunks[id] !== 0))) {
/******/ 				for(moduleId in moreModules) {
/******/ 					if(__webpack_require__.o(moreModules, moduleId)) {
/******/ 						__webpack_require__.m[moduleId] = moreModules[moduleId];
/******/ 					}
/******/ 				}
/******/ 				if(runtime) var result = runtime(__webpack_require__);
/******/ 			}
/******/ 			if(parentChunkLoadingFunction) parentChunkLoadingFunction(data);
/******/ 			for(;i < chunkIds.length; i++) {
/******/ 				chunkId = chunkIds[i];
/******/ 				if(__webpack_require__.o(installedChunks, chunkId) && installedChunks[chunkId]) {
/******/ 					installedChunks[chunkId][0]();
/******/ 				}
/******/ 				installedChunks[chunkId] = 0;
/******/ 			}
/******/ 		
/******/ 		}
/******/ 		
/******/ 		var chunkLoadingGlobal = self["webpackChunklearn_webpack_output"] = self["webpackChunklearn_webpack_output"] || [];
/******/ 		chunkLoadingGlobal.forEach(webpackJsonpCallback.bind(null, 0));
/******/ 		chunkLoadingGlobal.push = webpackJsonpCallback.bind(null, chunkLoadingGlobal.push.bind(chunkLoadingGlobal));
/******/ 	})();
```

只有当这个方法执行完成的时候，我们才知道 JSONP 成功与否，也就是script.onload/onerror 会在 webpackJsonpCallback 之后执行。所以 onload/onerror 其实是用来检查 webpackJsonpCallback 的完成度：有没有将 installedChunks 中对应的 chunk 值设为 0

## 如何打包vue文件

打包出来的vue文件和上述的js文件其实区别不大，只是vue文件会经过**vue-loader**处理，讲源码vue文件的内容转变成另一种形式。例如：

```javascript
<template>
    <div id="app">
      <!-- <img alt="Vue logo" src="./assets/logo.png"> -->
      <!-- <HelloWorld msg="Welcome to Your Vue.js App"/> -->
    </div>
  </template>
  
<script>
//   import HelloWorld from './components/HelloWorld.vue'
  
  export default {
    name: 'App',
  }
</script>
  
  <style>
  /* #app {
    font-family: Avenir, Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    text-align: center;
    color: #2c3e50;
    margin-top: 60px;
  } */
  </style>
  

```
被打包后

```javascript

"use strict";
__webpack_require__.r(__webpack_exports__);
/* harmony export */ __webpack_require__.d(__webpack_exports__, {
/* harmony export */   "default": () => (__WEBPACK_DEFAULT_EXPORT__)
/* harmony export */ });
/* harmony import */ var _App_vue_vue_type_template_id_7ba5bd90___WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(/*! ./App.vue?vue&type=template&id=7ba5bd90& */ "./src/App.vue?vue&type=template&id=7ba5bd90&");
/* harmony import */ var _App_vue_vue_type_script_lang_js___WEBPACK_IMPORTED_MODULE_1__ = __webpack_require__(/*! ./App.vue?vue&type=script&lang=js& */ "./src/App.vue?vue&type=script&lang=js&");
/* harmony import */ var _App_vue_vue_type_style_index_0_id_7ba5bd90_lang_css___WEBPACK_IMPORTED_MODULE_2__ = __webpack_require__(/*! ./App.vue?vue&type=style&index=0&id=7ba5bd90&lang=css& */ "./src/App.vue?vue&type=style&index=0&id=7ba5bd90&lang=css&");
/* harmony import */ var _node_modules_vue_loader_lib_runtime_componentNormalizer_js__WEBPACK_IMPORTED_MODULE_3__ = __webpack_require__(/*! !../node_modules/vue-loader/lib/runtime/componentNormalizer.js */ "./node_modules/vue-loader/lib/runtime/componentNormalizer.js");



;


/* normalize component */

var component = (0,_node_modules_vue_loader_lib_runtime_componentNormalizer_js__WEBPACK_IMPORTED_MODULE_3__["default"])(
  _App_vue_vue_type_script_lang_js___WEBPACK_IMPORTED_MODULE_1__["default"],
  _App_vue_vue_type_template_id_7ba5bd90___WEBPACK_IMPORTED_MODULE_0__.render,
  _App_vue_vue_type_template_id_7ba5bd90___WEBPACK_IMPORTED_MODULE_0__.staticRenderFns,
  false,
  null,
  null,
  null
  
)

/* hot reload */
if (false) { var api; }
component.options.__file = "src/App.vue"
/* harmony default export */ const __WEBPACK_DEFAULT_EXPORT__ = (component.exports);
```

可以看到一个vue打包后会变成引入template,script,style,componentNormalizer_js文件