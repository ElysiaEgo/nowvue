---
title: Vue2迁移到Vue3（一）
date: 2023/5/14 23:09:28
---
本系列主要围绕Vue2和Vue3的主要区别。

## 构建环境
`Vue CLI -> Vite`  
`Vuex -> Pinia`  
`Vetur -> Volar`  
`tsc -> vue-tsc`  

## setup语法糖
在Vue3中，为`SFC`引入了`<script setup>`，将`setup`从`export`中解放出来，优化了可读性。
```ts
<script setup>
// 变量
const msg = 'Hello!'

// 函数
function log() {
  console.log(msg)
}
</script>

<template>
  <button @click="log">{{ msg }}</button>
</template>
```
这样，`setup`会被识别成一个函数，所有在`<script setup>`的顶级变量会被uglify进一步缩小编译文件大小。（之前的处理方式会将所有顶级变量作为一个对象的属性）

## v-if & v-for
在Vue3中，v-if总是优先于v-for，与Vue2相反。

## 异步依赖（Suspense）
首先我们需要有一个异步加载的组件（setup为async）  
在上面提到的`<script setup>`中，我们只需要直接使用await，其setup就会变成async  
所有的异步组件都必须写在`<Suspense>`中，以下片段中，Dashboard本身或者它的子组件是异步组件，而`<template #fallback>`则是在加载中显示的内容
```ts
<Suspense>
  <!-- 具有深层异步依赖的组件 -->
  <Dashboard />

  <!-- 在 #fallback 插槽中显示 “正在加载中” -->
  <template #fallback>
    Loading...
  </template>
</Suspense>
```

