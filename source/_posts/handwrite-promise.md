---
title: 手写promise代码
tags: 
- js
- promise
categories: 
- 前端
---

let p1 = new Promise((resolve, reject) => {
    resolve('成功')
    reject('失败')
})
console.log('p1', p1)

let p2 = new Promise((resolve, reject) => {
    reject('失败')
    resolve('成功')
})
console.log('p2', p2)

let p3 = new Promise((resolve, reject) => {
    throw('报错')
})
console.log('p3', p3)


### 特性

1. 状态不可逆转，只能从pending 转移到其他状态
2. 支持链式调用
3. 提供resolve, reject，调用时执行相应的回调

``` javascript
class MyPromise{
    constructor(executor){
        this.initValue()
        this.initBind()
        try{
            executor(this.resovle, this.reject)
        }catch(e){
            this.reject(e)
        }
    }

    initValue(){
        this.status = 'pending'
        this.result = ''
        this.fullFilledList = []
        this.rejectedList = []
    }

    initBind(){
        this.resolve = this.resolve.bind(this)
    }

    resolve(val){
        if(this.status!=='pending')return
        this.status = 'success'
        this.fullFillList.shift()(val)
        this.result = val 
    }

    reject(val){
        if(this.status!=='pending')return
        this.status = 'reject'
        this.rejectedList.shift()(this.result)
        this.result = val
    }

    then(onFullFilled: (val)=>void, onRejcted: (val)=>void){
        var thenPromise = MyPromise((resolve,reject)=>{
            const resolvePromise = cb => {
                try{
                    const x = cb(this.PromiseResult)
                    if(x===thenPromise) throw Error('不能返回自身')
                }
                if(x instanceof MyPromise){
                    x.then(resove, reject)
                }else{
                    resolve(x)
                }
            } catch(err){
                reject(err)
            }
        })
        if(this.staus==='pending'){
            this.fullFilledList.push(onFullFilled)
            this.rejctedList.push(onRejcted)
            return thenPromise
        }
        if(this.status==='success'){
            resolvePromise(onFullFilled)
            return thenPromise
        }
        resolvePromise(onRejectd)

        return thenPromise
    }

    all(PromiseArr: MyPromise[]){
        return new MyPromise((resolve,reject)=>{       
            let count = 0
            let arr = []
            for(let i=0;i<PromiseArr.length;i++){
                PromiseArr[i].then((res)=>{
                    arr[i] = res
                    if(++count===PromiseArr.length){
                        resolve(arr)
                    }
                })
            }
        })
    }

    race(PromiseArr: MyPromise[]){
        return new MyPromise((resolve, reject)=>{
            for(let i=0;i<PromiseArr.length;i++){
                PromiseArr[i].then((res)=>{
                    resolve(res)
                })
            }
        })
    }
}
```

# 问题：

1. 如何获取多个promiseh获取结果？ 其中有一个promise报错怎么处理？

可以用promise.all，坏处是其中有一个报错的话那么其他成功的promise结果就不会返回，解决方案可以换成promise.alllSettled。
另外也可以对promose数组里面的一promise.catch处理，使其报错不影响返回。