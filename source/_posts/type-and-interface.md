---
title: Typescript如何限制对象的key值
tags: 
- js
- promise
categories: 
- 前端
---

我们有类似以下的数据
```ts
const students = {
    'freshman': 1
    'sophomore':2,
    'junior': 3,
    'senior':4
}
```

这个是类似于写死的配置，并且只能从特定的key获取值。 我们可以这样写，先定义一个枚举

```ts
enum Grade {
    FreshMan = 'freshman',
    Sophomore = 'sophomore',
    Sophomore = 'junior',
    Senior = 'senior'
}
```

然后再定义

```js
type Students =  Record<Grade, number>
```

定义了枚举之后我们的配置数据可以写成这样

```ts
const students = {
    [Grade.FreshMan]: 1
    [Grade.Sophomore]:2,
    [Grade.Sophomore]: 3,
    [Grade.Senior]:4
}
```

### 2. keyof 和 in 的区别

keyof: 取interface的键后保存为联合类型

```typescript
interface userInfo {
  name: string
  age: number
}
type keyofValue = keyof userInfo
// keyofValue = "name" | "age"

```

in: 取联合类型的值，主要用于数组和对象的构建

```typescript
type name = 'firstname' | 'lastname'
type TName = {
  [key in name]: string
}
// TName = { firstname: string, lastname: string }
```
