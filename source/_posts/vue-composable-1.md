---
title: 深入Vue响应式原理（一）
date: 2023/5/1 12:45:30
---
>Vue 最标志性的功能就是其低侵入性的响应式系统。组件状态都是由响应式的 JavaScript 对象组成的。当更改它们时，视图会随即自动更新。这让状态管理更加简单直观。
Vue的响应式可以轻松降低编码难度，但是使用不当也会造成很多麻烦，本篇文章以```computed```为例将分析Vue是如何监听数据变化并更新组件的。

## 为什么要采取响应式
前端通常是获取后端数据，进行一定程度的处理，再进行渲染。这其中涉及到了网络请求中，处理中和处理完成的状态。在较为复杂的页面中为每个与后端交流的请求处理这三种状态是重复机械的工作。  
而响应式将这种状态抽象出来，通过监听数据变化，再调用处理函数，最后进行渲染，使编码的重点集中在了如何处理数据和渲染。

## 监听数据变化
js原生不支持追踪基本数据类型的读写，但是通过```Proxy```可以实现追踪对象属性的读写。（只争对Vue3，Vue2不作讨论）  
下面是来自Vue的伪代码。
```js
function ref(value) {
  const refObject = {
    get value() {
      track(refObject, 'value')
      return value
    },
    set value(newValue) {
      value = newValue
      trigger(refObject, 'value')
    }
  }
  return refObject
}
```
以上代码通过重写Proxy对象的setter/getter来触发处理。
```js
import { ref, computed } from 'vue'

const A0 = ref(0)
const A1 = ref(1)
const A2 = computed(() => A0.value + A1.value)

A0.value = 2
```
在上面的代码中，对A0.value赋值触发了trigger，进而使computed中原来的值过期，触发重新计算，进而触发重新渲染，最终达到更新页面内容的目的。

## 简单应用-SWR
```ts
import { ref, Ref, UnwrapRef } from 'vue'

export interface Result<T> {
  data: Ref<UnwrapRef<T | null>>
  error: Ref<Error | null>
  loading: Ref<boolean>
}

export const useRequest = <T>(promise: Promise<T>, initialData: T | null = null): Result<T> => {
  const data = ref<T | null>(initialData)
  const error = ref<Error | null>(null)
  const loading = ref<boolean>(true)

  promise.then(
    val => {
      data.value = val as UnwrapRef<T>
      loading.value = false
    },
    err => {
      error.value = err
      loading.value = false
    }
  )

  return { data, error, loading }
}
```
在上面的例子中，```useRequest```接受两个参数，一个为Promise即网络请求，第二个是初始化数据，即没有完成网络请求之前的填充数据。  
随后附加then到Promise上，在Promise完成时将result.data设置为Promise的返回值，并设置loading为false，错误处理也是同理。  
使用时，只需要将```useRequest```返回值用ref包含，然后在```computed```中处理，即可在组件中使用数据。  
下面是一个简单的获取，处理数据，渲染的例子
```ts
<template>
  <n-card>
    <n-data-table
      :columns="createColumns()"
      :data="OrderOptions"
      :bordered="false"
    />
  </n-card>
</template>

<script lang="ts" setup>
import { CascaderOption } from 'naive-ui'
import { computed, ref } from 'vue'
import { Order, getOrder } from '../api/order'
import { useRequest } from '../composables/request'

const orders = ref(useRequest(getOrder()))
const OrderOptions = computed(() => (orders.value.data?.orders.length === 0
  ? []
  : orders.value.data?.orders.map((value: Order, index) => {
    return {
      no: index,
      finish: value.finish ? '√' : 'x',
      message: value.message === null ? '-' : value.message,
    }
  })))
</script>
```
