---
title: Vue3基础（二）
date: 2024/10/18 19:00:00   
tags: 
- vue3
categories: 
- 前端
---

下面是vue3一些常见的api

# defineComponent

在定义 Vue 组件时提供类型推导的辅助函数。即： 让传入的整个对象获得对应的类型，**在TypeScript下，给予了组件 正确的参数类型推断。**

```javascript
// 选项语法
function defineComponent(
  component: ComponentOptions
): ComponentConstructor

// 函数语法 (需要 3.3+)
function defineComponent(
  setup: ComponentOptions['setup'],
  extraOptions?: ComponentOptions
): () => any
```

详细信息

第一个参数是一个组件选项对象。返回值将是该选项对象本身，因为该函数实际上在运行时没有任何操作，仅用于提供类型推导。

注意返回值的类型有一点特别：它会是一个构造函数类型，它的实例类型是根据选项推断出的组件实例类型。这是为了能让该返回值在 TSX 中用作标签时提供类型推导支持。

你可以像这样从 defineComponent() 的返回类型中提取出一个组件的实例类型 (与其选项中的 this 的类型等价)：

# setup

在vue3中没有this对象, 所有的逻辑代码都写在setup方法里面, 若是要在HTML模板页面中使用变量或者方法, 需要在setup方法return出去。
也可以这么写

```JS
<script setup>
const a = ref('1')
const handleChange = ()=>{
  console.log(a.value)
}
</srcipt>
```

不用写return, template中就能写任何在script中定义好的变量和函数。

# defineExpose

defineExpose 是Vue3中的一个新API，它允许子组件暴露其内部属性或方法给父组件访问。可以通过将属性或方法添加到defineExpose函数中来实现。

获取用setup语法糖创建的子组件实例时，获取的实例是没有子组件自定义的属性和方法的，此时我们需要通过defineExpose来暴露子组件的属性和方法。

1、定义子组件 

```javascript
<template>
   <p>子组件test</p>
</template>

<script setup>
import { ref } from 'vue'

const msg = ref('Hello Vue3')

const a = () => {
  console.log(1)
}

const handleChangeMsg = (v) => { 
  msg.value = v 
}

defineExpose({
    msg, 
    a, 
    handleChangeMsg 
})
</script>
```

2. 定义父组件
```javascript

<template>
   <Test ref="testInstanceRef" />
   <button @click="handleChangeMsg">handleChangeMsg</button>
</template>

<script setup>
import { ref, onMounted } from "vue";
import Test from "./components/Test.vue";

const testInstanceRef = ref();

onMounted(() => {
   const testInstance = testInstanceRef.value;
   console.log(testInstance.$el)   // p标签
   console.log(testInstance.msg)   // msg属性
   console.log(testInstance.a)    // a方法，如果不使用defineExpose暴露是拿不到的
})

const handleChangeMsg = () => { 
   testInstanceRef.value.handleChangeMsg('Hello TS') 
}
</script>

```

# 获取this

Vue2 中每个组件里使用 this 都指向当前组件实例，this 上还包含了全局挂载的东西、路由、状态管理等啥啥都有
而 Vue3 组合式 API 中没有 this，如果想要类似的用法，有两种，一是获取当前组件实例，二是获取全局实例，如下自己可以去打印出来看看

```javascript

<script setup>
import { getCurrentInstance } from 'vue'

// proxy 就是当前组件实例，可以理解为组件级别的 this，没有全局的、路由、状态管理之类的
const { proxy, appContext } = getCurrentInstance()!

// 这个 global 就是全局实例
const global = appContext.config.globalProperties
</script>

```

# 全局注册(属性/方法)

Vue2 中我们要往全局上挂载东西通常就是如下，然后在所有组件里都可以通过 this.xxx 获取到了

```javascript
Vue.prototype.xxx = xxx
```

而 Vue3 中不能这么写了，换成了一个能被所有组件访问到的全局对象，就是上面说的全局实例的那个对象，比如在 main.js 中做全局注册

```javascript

// main.js
import { createApp } from 'vue'
import App from './App.vue'
const app = createApp(App)
// 添加全局属性
app.config.globalProperties.name = '沐华'

```

在组件中调用

```javascript

<script setup>
import { getCurrentInstance } from 'vue'
const { appContext } = getCurrentInstance()

const global = appContext.config.globalProperties
console.log(global.name)
</script>

```

# 获取 DOM

```javascript
<template>
    <el-form ref="formRef"></el-form>
    <child-component />
</template>
<script setup lang="ts">
import ChildComponent from './child.vue'
import { getCurrentInstance } from 'vue'
import { ElForm } from 'element-plus'

// 方法一，这个变量名和 DOM 上的 ref 属性必须同名，会自动形成绑定
const formRef = ref(null)
console.log(formRef.value) // 这就获取到 DOM 了

// 方法二
const { proxy } = getCurrentInstance()
proxy.$refs.formRef.validate((valid) => { ... })

// 方法三，比如在 ts 里，可以直接获取到组件类型
// 可以这样获取子组件
const formRef = ref<InstanceType<typeof ChildComponent>>()
// 也可以这样 获取 element ui 的组件类型
const formRef = ref<InstanceType<typeof ElForm>>()
formRef.value?.validate((valid) => { ... })
</script>

```


# 初始化

Vue2 中进入页面就请求接口，或者其他一些初始化的操作，一般放在 created 或 mounted，而 Vue3 中 beforeCreated 和 created 这俩钩子就不用了，因为 setup 在这俩之前执行，还要这俩的话就多此一举了
所以以前用在 beforeCreated / created / beforeMount / mounted 这几个钩子里的内容，在 Vue3 中可以直接放在 setup 里，或者放在 onMounted/onBeforeMount 里

```javascript

<script setup>
import { onMounted } from 'vue'

// 请求接口函数
const getData = () => {
    xxxApi.then(() => { ... })
}

onMounted(() => {
    getData()
})
</script>

```

# watch

在vue2我们一般使用watch去监听数据的更改，然后执行相应的函数

```javascript
watch: {
    userName(newVal, oldVal) {
        this.handleNewUserName()
    },
    userInfo:{
        handler: (newVal, oldVal)=> {this.getData()},
        immediate: true
        deep: true
    }
}

```

Vue3 的 watch 是一个函数，能接收三个参数，参数一是监听的属性，参数二是接收新值和老值的回调函数，参数三是配置项


```javascript

<script lang="ts" setup>
import {watch， ref} from 'vue'

const userInfo = ref({name:'xx', age: 18})

watch(() => userInfo.age, (newVal, oldVal)=>{})


// 监听多个属性，数组放多个值，返回的新值和老值也是数组形式
watch([data.age, data.money], ([newAge, newMoney], [oldAge, oldMoney]) => { ... })


// 第三个参数是一个对象，为可配置项，有5个可配置属性
watch(data.children, (newList, oldList) => { ... }, {
    // 这两个和 Vue2 一样，没啥说的
    immediate: true,
    deep: true,
    // 回调函数的执行时机，默认在组件更新之前调用。更新后调用改成post
    flush: 'pre', // 默认值是 pre，可改成 post 或 sync
    // 下面两个是调试用的
    onTrack (e) { debugger }
    onTrigger (e) { debugger }
})

</script>

```

# watchEffect

```javascript
// 监听方法赋值
const unwatch = watch('key', callback)
const unwatchEffect = watchEffect(() => {})
// 需要停止监听的时候，手动调用停止监听
unwatch()
unwatchEffect()
```