---
title: New to Vuex
date: 2023/4/29 01:54:09
---
起因是在项目中有token api，为了统一管理token，遂采取  
```typescript
export const state = reactive({})
```
这样的方式缺失了ts的类型约束， 同时更新属性时非常麻烦。  
而Vuex就是为了解决这些问题，同时为ssr提供了支持。  
Vuex是这样介绍自己的  
> Vuex 是一个专为 Vue.js 应用程序开发的状态管理模式 + 库。它采用集中式存储管理应用的所有组件的状态，并以相应的规则保证状态以一种可预测的方式发生变化。

## Quick Start
一个简单的实例，定义一个count，同时有一个自增函数  
```typescript
import { createApp } from 'vue'
import { createStore } from 'vuex'
import App from './app.vue'

// 创建一个新的 store 实例
const store = createStore({
  state () {
    return {
      count: 0
    }
  },
  mutations: {
    increment (state) {
      state.count++
    }
  }
})

const app = createApp(App)

// 将 store 实例作为插件安装
app.use(store)
```

后续在其他组件中，通过this.$store可以访问count并自增  
```typescript
this.$store.commit('increment')
console.log(this.$store.state.count)
```

## Further more
上面我们解决了储存的问题，并保留了ts的类型提示，但是token不会一次分发，而是有expires，需要刷新  
为了实现刷新，Vuex提供了修改state的api```Mutation```，也就是上面的自增  
可惜的是```Mutation```只能是同步函数，刷新token需要与服务器通信，是异步过程  
Vuex也提供了对应的api```Action```
同样以上面的实例为基础， 模拟async获取token过程  
```typescript
const store = createStore({
  state: {
    token: 'aaa'
  },
  mutations: {
    update (state, payload) {
      state.token = payload.newToken
    }
  },
  actions: {
    async refresh (context) {
      const refreshToken = await new Promise<string>((resolve, reject) => {
        resolve('token')
      })
      context.commit('update', refreshToken)
    }
  }
})
```
这时候，当我们需要刷新token的时候只需要执行下面的代码
```typescript
await store.dispatch('refresh', { newToken: '' }).then(() => {
  console.log(store.state.token)
})
```
