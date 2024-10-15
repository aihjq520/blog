---
title: 大文件下载与promise相关知识
tags: 
- js
- promise
categories: 
- 前端
---
# 大文件下载（Promise相关知识）
## 1.Promise

Promise对象是一个构造函数，用来生成Promise实例。
下面代码创造了一个Promise实例。
```javascript
const promise = new Promise(function(resolve, reject) {
  // ... some code
  if (/* 异步操作成功 */){
    resolve(value);
  } else {
    reject(error);
  }
});
```

下面是一个Promise对象的简单例子。
```javascript
function timeout(ms) {
  return new Promise((resolve, reject) => {
    setTimeout(resolve, ms, 'done');
  });
}
```
```javascript
timeout(100).then((value) => {
  console.log(value);
});
```
上面代码中，timeout方法返回一个Promise实例，表示一段时间以后才会发生的结果。过了指定的时间（ms参数）以后，Promise实例的状态变为resolved，就会触发then方法绑定的回调函数。


Promise 新建后就会立即执行。
```javascript
let promise = new Promise(function(resolve, reject) {
  console.log('Promise');
  resolve();
});
promise.then(function() {
  console.log('resolved.');
});
console.log('Hi!');
// Promise
// Hi!
// resolved
```
上面代码中，Promise 新建后立即执行，所以首先输出的是Promise。然后，then方法指定的回调函数，将在当前脚本所有同步任务执行完才会执行，所以resolved最后输出。


## 2.大文件下载
### 1.Range
Range是在 HTTP/1.1 中新增的一个字段，这个特性也是我们使用的迅雷等支持多线程下载以及断点下载的核心机制。
首先客户端会发起一个带有Range: bytes=0-xxx的请求，如果服务端支持 Range，则会在响应头中添加Accept-Ranges: bytes来表示支持 Range 的请求，之后客户端才可能发起带 Range 的请求。

ange的格式为：
Range:(unit=first byte pos)-[last byte pos]
即Range: 单位（如bytes）= 开始字节位置-结束字节位置。
我们来举个例子，假设我们开启了多线程下载，需要把一个5000byte的文件分为4个线程进行下载。

Range: bytes=0-1199 头1200个字节
Range: bytes=1200-2399 第二个1200字节
Range: bytes=2400-3599 第三个1200字节
Range: bytes=3600-5000 最后的1400字节

服务器给出响应：
第1个响应

Content-Length：1200
Content-Range：bytes 0-1199/5000

第2个响应

Content-Length：1200
Content-Range：bytes 1200-2399/5000

第3个响应

Content-Length：1200
Content-Range：bytes 2400-3599/5000

第4个响应

Content-Length：1400
Content-Range：bytes 3600-5000/5000

如果每个请求都成功了，服务端返回的response头中有一个 Content-Range 的字段域，Content-Range 用于响应头，告诉了客户端发送了多少数据，它描述了响应覆盖的范围和整个实体长度。一般格式：
Content-Range: bytes (unit first byte pos) - [last byte pos]/[entity length]即Content-Range：字节 开始字节位置-结束字节位置／文件大小。

### 2.Range实践
![](https://s3.qiufengh.com/blog/1600705973008.jpg)


参考文章
- https://juejin.cn/post/6874991690747215880
