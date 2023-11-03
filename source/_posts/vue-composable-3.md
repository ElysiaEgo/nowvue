---
title: 深入Vue响应式原理（三）
date: 2023/11/3 10:27:30
---
# Vue是如何创建`Ref`的
>在阅读这篇博客之前，建议至少阅读这个系列的[第一篇](https://blog.elysiaego.top/2023/05/07/vue-composable-2/)。本篇博客假设你已经懂得使用Vue中的ref和TypeScript基础。

本篇博客深入源码，通过日常使用的底层执行顺序进行分析，使你理解响应式原理。  
以下代码中，中文注释是本人创作，请遵守CC BY-NC-SA 4.0协议。英文代码均为Vue源码，请遵守Vue的MIT开源协议。

源码：  
[src/v3/reactivity/ref.ts](https://github.com/vuejs/vue/blob/main/src/v3/reactivity/ref.ts)  
[src/core/observer/dep.ts](https://github.com/vuejs/vue/blob/main/src/core/observer/dep.ts)  

```typescript
// Ref的类型定义
export interface Ref<T = any> {
  // 一切关键的关键
  value: T
  /**
   * Type differentiator only.
   * We need this to be in public d.ts but don't want it to show up in IDE
   * autocomplete, so we use a private Symbol instead.
   */
  [RefSymbol]: true
  /**
   * @internal
   */
  dep?: Dep
  /**
   * @internal
   */
  [RefFlag]: true
}
export type ShallowRef<T = any> = Ref<T> & { [ShallowRefMarker]?: true }
// 浅Ref和Ref创建函数
export function ref(value?: unknown) { return createRef(value, false) }
export function shallowRef(value?: unknown) { return createRef(value, true) }

// 创建Ref的关键函数
function createRef(rawValue: unknown, shallow: boolean) {
  // 不能在ref上Ref，直接返回ref
  if (isRef(rawValue)) {
    return rawValue
  }
  const ref: any = {}
  // 这两个字段是Vue内部使用，不需要管
  def(ref, RefFlag, true)
  def(ref, ReactiveFlags.IS_SHALLOW, shallow)
  // 定义这个Ref的订阅，以便在ref触发修改时更新
  def(
    ref,
    'dep',
    // 关键的value定义就在这里！
    defineReactive(ref, 'value', rawValue, null, shallow, isServerRendering())
  )
  return ref
}

// 在对象上定义属性，工具函数
export function def(obj: Object, key: string, val: any, enumerable?: boolean) {
  Object.defineProperty(obj, key, {
    value: val,
    // 当enumerable为undefined时，两次取反为false
    enumerable: !!enumerable,
    writable: true,
    // configurable (from MDN)
    // 当设置为 false 时，该属性的类型不能在数据属性和访问器属性之间更改，且该属性不可被删除，且其描述符的其他属性也不能被更改（但是，如果它是一个可写的数据描述符，则 value 可以被更改，writable 可以更改为 false）。
    configurable: true
  })
}

// 响应式的关键函数，只需要关注前三个参数
// 负责创建Dep订阅，并添加value字段
export function defineReactive(
  obj: object,// ref
  key: string,// value
  val?: any, // rawValue
  customSetter?: Function | null,
  shallow?: boolean,
  mock?: boolean
) {
  const dep = new Dep()

  // 如果dep已经定义且不是configurable则中止
  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // NO_INITIAL_VALUE是常量，值为{}，在创建响应式前创建一个默认值，防止无参数的ref()
  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set
  if (
    (!getter || setter) &&
    (val === NO_INITIAL_VALUE || arguments.length === 2)
  ) {
    val = obj[key]
  }

  let childOb = !shallow && observe(val, false, mock)
  // 为value绑定getter/setter，通过.value访问对象时，就会触发。
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter() {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        if (__DEV__) {
          dep.depend({
            target: obj,
            type: TrackOpTypes.GET,
            key
          })
        } else {
          // 对对象整体进行代理
          dep.depend()
        }
        // 非浅ref则对对象的字段也进行代理
        if (childOb) {
          childOb.dep.depend()
          if (isArray(value)) {
            dependArray(value)
          }
        }
      }
      // 递归解包Ref
      return isRef(value) && !shallow ? value.value : value
    },
    set: function reactiveSetter(newVal) {
      // 获取当前值
      const value = getter ? getter.call(obj) : val
      // 比较当前值和新值
      if (!hasChanged(value, newVal)) {
        return
      }
      if (__DEV__ && customSetter) {
        customSetter()
      }
      // 设置新值
      if (setter) {
        setter.call(obj, newVal)
      } else if (getter) {
        // #7981: for accessor properties without setter
        return
      } else if (!shallow && isRef(value) && !isRef(newVal)) {
        value.value = newVal
        return
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal, false, mock)
      // 触发订阅，执行副作用
      if (__DEV__) {
        dep.notify({
          type: TriggerOpTypes.SET,
          target: obj,
          key,
          newValue: newVal,
          oldValue: value
        })
      } else {
        dep.notify()
      }
    }
  })

  return dep
}

// 实现依赖订阅的类
/**
 * A dep is an observable that can have multiple
 * directives subscribing to it.
 * @internal
 */
export default class Dep {
  // 注意static，为了保证全局同一时间只运行一个副作用，target会固定为当前副作用的target
  static target?: DepTarget | null
  id: number
  subs: Array<DepTarget | null>
  // pending subs cleanup
  _pending = false

  constructor() {
    // uid是Dep模块静态字段，保证全局Dep id唯一
    this.id = uid++
    // 订阅者合集
    this.subs = []
  }

  // 添加订阅者
  addSub(sub: DepTarget) {
    this.subs.push(sub)
  }
  
  // 移除订阅者
  removeSub(sub: DepTarget) {
    // #12696 deps with massive amount of subscribers are extremely slow to
    // clean up in Chromium
    // to workaround this, we unset the sub for now, and clear them on
    // next scheduler flush.
    this.subs[this.subs.indexOf(sub)] = null
    if (!this._pending) {
      this._pending = true
      pendingCleanupDeps.push(this)
    }
  }

  depend(info?: DebuggerEventExtraInfo) {
    if (Dep.target) {
      // 注意this的指向
      Dep.target.addDep(this)
      if (__DEV__ && info && Dep.target.onTrack) {
        Dep.target.onTrack({
          effect: Dep.target,
          ...info
        })
      }
    }
  }

  // 给每个订阅者触发副作用，面试可能会问到副作用的执行顺序
  notify(info?: DebuggerEventExtraInfo) {
    // stabilize the subscriber list first
    const subs = this.subs.filter(s => s) as DepTarget[]
    if (__DEV__ && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      const sub = subs[i]
      if (__DEV__ && info) {
        sub.onTrigger &&
          sub.onTrigger({
            effect: subs[i],
            ...info
          })
      }
      sub.update()
    }
  }
}
```
## 习题
1. 尝试使用原生js，给对象设置getter/setter，在其中更新dom

## 下集预告
1. computed
2. Dep.depend

## 扩展阅读
[get](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Functions/get)
[set](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Functions/set)
[Proxy](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy)
[ecma-262/es6 规范](https://github.com/tc39/ecma262)
