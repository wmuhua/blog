---
title: Vue3.2响应式源码剖析及与Vue2的区别
date: 2020-09-08 06:49:47
author: 沐华
img: https://cdn.jsdelivr.net/gh/wmuhua/cdn@main/blog/25.jpg
top: false
cover: false
coverImg: /images/1.jpg
password:
toc: false
mathjax: false
summary: 我们知道相较 Vue2.x 的响应式 Vue3 对整个响应式都做了重大升级；然后 Vue3.2 相较 3.0 版本源码又做了许多变更，一起来看看吧
categories: Vue
tags:
  - Vue3
  - 响应式
  - 源码
---

本文源码版本 `Vue3.2.11`，Vue2 响应式源码剖析点这里 [深入浅出 Vue2 响应式原理源码剖析](https://juejin.cn/editor/drafts/7009900485691834398)

我们知道相较 Vue2.x 的响应式 Vue3 对整个响应式都做了重大升级；然后 Vue3.2 相较 3.0 版本源码又做了许多变更，一起来看看吧

## Vue3 和 Vue2 响应式区别

### 响应式性能的提升

根据 8 月 10 号尤大发布 [Vue3.2 说明原文](https://blog.vuejs.org/posts/vue-3.2.html) 得知：

- 更高效的 `ref` 实现，读取提升约 `260%`，写入提升约 `50%`
- 依赖收集速度提升约 `40%`
- 减少内存消耗约 `17%`

### 使用上的区别

Vue2 中只要写在组件中 data 函数返回的对象里的属性 自动就有响应式

Vue3 则是通过 ref 定义普通类型响应式和 reactive 定义复杂类型响应式数据

```js
<script setup>
  import { ref, reactive, toRefs } from "vue"
  const name = ref('沐华')
  const obj = reactive({ name: '沐华' })
  const data = { ...toRefs(obj) }
</script>
```

> 扩展：通过 toRefs 可以把响应式对象转为普通对象  
> 因为使用 reactive 定义的响应式对象在进行解构(展开)或者销毁的时候，响应式就会失效了，因为 reactive 实例下有很多属性，解构就丢失了，所以在需要解构且保持响应式的时候就可以用 toRefs

### 源码目录结构区别

Vue2 响应式的源码核心部分在 `src/core/observer` 这个目录，但是里面也引入了很多其他目录东西，不独立，耦合度比较高

Vue3 响应式源码全部在 `packages/reactivity` 这个目录下，不涉及其他任何地方，功能独立，而且单独发布成 npm 包，可以集成进其他框架

### 原理上的区别

我们知道在 Vue2 中使用 `Object.defineProperty` 实现响应式对象，而这种方式是存在一些缺陷的

- 基于属性拦截，初时化时会递归全部属性，对性能有一定影响，并且后续给对象中添加的新属性，无法触发响应式，对应的解决办法是通过 `Vue.set()` 方法来添加新属性
- 无法检测到数组内部变化，对应的解决方法是通过重写了 7 个会改变原数组的方法

而在 Vue3 中则是用 `Proxy` 进行重构，完全取代了 `defineProperty`，就不存在上述问题了

那么是如何解决这些问题的呢？

#### 对象

先看下 Vue2 在首次渲染时的响应式处理，源码地址：`src/core/observer/index.js - 157行`

```
...
Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () { ... },
    set: function reactiveSetter (newVal) { ... }
})
...
```

由参数可以看出，它是需要根据具体的 key 去 obj 里找 `obj[key]`，来进行拦截处理的，所以就有需要满足一个前置条件，一开始就得知道 key 是啥，所以就需要遍历每一个 key，并定义 `getter`、`setter`，这也是为什么后面添加的属性没有响应式的原因

而 Vue3 中则是这样的

```js
// ref 源码      `packages/reactivity/ref.ts -142行`
// reactive 源码 `packages/reactivity/reactive.ts -173行`
new Proxy(target, {
  // target 为组件的 data 返回的对象
  get(target, key) {},
  set(target, key, value) {},
})
```

同样由参数就可以看出，开始创建响应式的时候，根本不需要知道这个对象里有哪些字段，因为不用传具体的 key，这样就算是后面新增的，自然也能够拦截得到

也就是说不会上来就递归遍历把所有用到没用到的都设置响应式，从而加快了首次渲染

#### 数组

在 Vue2 中

- 一个是因为 `Object.defineProperty` 这个 api 无法监听到数组长度的变化
- 二是因为数组长度可能很长，比如 `lenth` 是大几千，上万的，所以尤大考虑到性能消耗与用户体验，设计的就是 Vue 本身就不能监听直接通过下标修改数组元素的操作

延伸一个问题，为什么无法监听到数组长度的变化呢？先看图

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/50776e1ed8ab4db0848e351842c2f76f~tplv-k3u1fbpfcp-watermark.image?)

如图就是 `configurable` 为 true 时对应的值才能被改变，也可以理解成才能被监听，而 length 本身是不可以被监听的，所以数组长度改变时也监听不到

如果强行把它的 configurable 修改为 true 则会报错，因为各大浏览器厂商和 JS 引擎规定就不允许修改 length 的 configurable，规定就是这样，所以在源码里才会有这样的代码

```js
// `src/core/observer/index.js - 144行`
const property = Object.getOwnPropertyDescriptor(obj, key)
if (property && property.configurable === false) {
  return
}
```

所以为了更好的操作数组并触发响应式，就重写了会改变原数组的 7 个方法，再通过 `ob.dep.notify()` 手动派发更新，源码地址：`src/core/observer/array.js`

在 Vue3 中使用 Proxy，Proxy 就是代理的意思，回顾一下语法

```js
new Proxy(target, {
  get(target, key) {},
  set(target, key, value) {},
})
```

根据 [MDN 中对 Proxy 的描述](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy) 是这样的

- `target`: 被 Proxy 代理虚拟化的对象。它常被作为代理的存储后端。根据目标验证关于对象**不可扩展性或不可配置属性**的不变量（保持不变的语义）

注意了：数组的 length 就是不可配置的属性，所以 Proxy 天生就能监听数组长度变化

### 依赖收集的区别

Vue2 中是通过 `Observer`，`Dep`，`Watcher` 这三个类来实现依赖收集，详细流程可以看我另一篇文章[ 深入浅出 Vue 响应式原理源码剖析](https://juejin.cn/post/7017327623307001864)

Vue3 中是通过 `track` 收集依赖，通过 `trigger` 触发更新，本质上就是用 WeakMap，Map，Set 来实现，具体可以看下面源码的实现过程

### 缺点区别

上面也有提到 Vue2 中 `defineProperty` 监听不到新增对象属性/数组内部变化，而且属性值是对象的话会多次调用 `observe()` 递归遍历，还有就是会对所有数据属性都设置监听，包括没有用到的属性，性能上自然就没那么好

在 Vue3 中主要就是大量使用 Es6+ 新特性，在老版本浏览器上兼容就没那么好

## Vue3 响应式源码解析

先看一下在 Vue3 中定义的几个用来标记目标对象 target 的类型的 flag，下面先是枚举的属性

源码地址：`packages/reactivity/reactive.ts -16行`

```js
export const enum ReactiveFlags {
  SKIP = '__v_skip',
  IS_REACTIVE = '__v_isReactive',
  IS_READONLY = '__v_isReadonly',
  RAW = '__v_raw'
}
export interface Target {
  [ReactiveFlags.SKIP]?: boolean // 不做响应式处理的数据
  [ReactiveFlags.IS_REACTIVE]?: boolean // target 是否是响应式
  [ReactiveFlags.IS_READONLY]?: boolean // target 是否是只读的
  [ReactiveFlags.RAW]?: any // 表示 proxy 对应的源数据，target 已经是 proxy 对象时会有该属性
}
```

然后开始一一解析

### reactive()

源码地址：`packages/reactivity/src/reactive.ts -87行`

```js
export function reactive<T extends object>(target: T): UnwrapNestedRefs<T>
export function reactive(target: object) {
  // 如果 target 是只读类型的对象就直接返回
  if (target && (target as Target)[ReactiveFlags.IS_READONLY]) {
    return target
  }
  return createReactiveObject(
    target, // 需要创建响应式的目标对象 data
    false, // 不是只读类型
    mutableHandlers,
    mutableCollectionHandlers,
    reactiveMap // const reactiveMap = new WeakMap<Target, any>()
  )
}
```

这里代码很简单，主要就是调用 createReactiveObject()

### createReactiveObject()

源码地址：`packages/reactivity/src/reactive.ts -173行`

```js
function createReactiveObject(
  target: Target,
  isReadonly: boolean,
  baseHandlers: ProxyHandler<any>,
  collectionHandlers: ProxyHandler<any>,
  proxyMap: WeakMap<Target, any>
) {
  // typeof 不是 object 类型的，直接返回
  if (!isObject(target)) {
    if (__DEV__)
      console.warn(`value cannot be made reactive: ${String(target)}`)
    return target
  }
  // 已经是响应式的就直接返回
  if (
    target[ReactiveFlags.RAW] &&
    !(isReadonly && target[ReactiveFlags.IS_REACTIVE])
  ) {
    return target
  }
  // 如果已经存在 map 中了，就直接返回
  const existingProxy = proxyMap.get(target)
  if (existingProxy) {
    return existingProxy
  }
  // 不做响应式的，直接返回
  const targetType = getTargetType(target)
  if (targetType === TargetType.INVALID) {
    return target
  }
  // 把 target 转为 proxy
  const proxy = new Proxy(
    target,
    targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers
  )
  // 添加到 map 里
  proxyMap.set(target, proxy)
  return proxy
}
```

大概了解了这个方法里要做的事，接下来我们还要先明白传入的几个参数是什么

参数配置定义是这样的

```js
const get = /*#__PURE__*/ createGetter()
const set = /*#__PURE__*/ createSetter()
export const mutableHandlers: ProxyHandler<object> = {
  get, // 获取属性
  set, // 修改属性
  deleteProperty, // 删除属性
  has, // 是否拥有某个属性
  ownKeys, // 收集 key，包括 symbol 类型或者不可枚举的 key
}
```

这里 get、has、ownKeys 会触发依赖收集 track()
set、deleteProperty 会触发更新 trigger()

其中有两个重要的方法就是 get 和 set 对应的 createGetter 和 createSetter

### createGetter()

源码地址：`packages/reactivity/src/baseHandlers.ts -80行`

```js
function createGetter(isReadonly = false, shallow = false) {
  return function get(target: Target, key: string | symbol, receiver: object) {
    // 访问对应标记位
    if (key === ReactiveFlags.IS_REACTIVE) {
      return !isReadonly
    } else if (key === ReactiveFlags.IS_READONLY) {
      return isReadonly
    } else if (
      // receiver 指向调用者，这里判断是为了保证触发拦截 handle 的是 proxy 本身而不是 proxy 的继承者
      // 触发拦的两种方式：一是访问 proxy 对象本身的属性，二是访问对象原型链上有 proxy 对象的对象的属性，因为查询会沿着原型链向下找
      key === ReactiveFlags.RAW &&
      receiver ===
        (isReadonly
          ? shallow
            ? shallowReadonlyMap
            : readonlyMap
          : shallow
          ? shallowReactiveMap
          : reactiveMap
        ).get(target)
    ) {
      // 返回 target 本身，也就是响应式对象的原始值
      return target
    }
    // 是否是数组
    const targetIsArray = isArray(target)
    // 不是只读类型 && 是数组 && 触发的是 arrayInstrumentations 工具集里的方法
    if (!isReadonly && targetIsArray && hasOwn(arrayInstrumentations, key)) {
      // 通过 proxy 调用，arrayInstrumentations[key]的this一定指向 proxy
      return Reflect.get(arrayInstrumentations, key, receiver)
    }
    // proxy 预返回值
    const res = Reflect.get(target, key, receiver)
    // key 是 symbol 或访问的是__proto__属性不做依赖收集和递归响应式处理，直接返回结果
    if (isSymbol(key) ? builtInSymbols.has(key) : isNonTrackableKeys(key)) {
      return res
    }
    // 不是只读类型的 target 就收集依赖。因为只读类型不会变化，无法触发 setter，也就会触发更新
    if (!isReadonly) {
      // 收集依赖，存储到对应的全局仓库中
      track(target, TrackOpTypes.GET, key)
    }
    // 浅比较，不做递归转化，就是说对象有属性值还是对象的话不递归调用 reactive()
    if (shallow) {
      return res
    }
    // 访问的属性已经是 ref 对象
    if (isRef(res)) {
      // 返回 ref.value，数组除外
      const shouldUnwrap = !targetIsArray || !isIntegerKey(key)
      return shouldUnwrap ? res.value : res
    }
    // 由于 proxy 只能代理一层，如果子元素是对象，需要递归继续代理
    if (isObject(res)) {
      return isReadonly ? readonly(res) : reactive(res)
    }

    return res
  }
}
```

track() 依赖收集放到后面，和派发更新一起

### createSetter()

源码地址：`packages/reactivity/src/baseHandlers.ts -80行`

```js
function createSetter(shallow = false) {
  return function set(
    target: object,
    key: string | symbol,
    value: unknown,
    receiver: object
  ): boolean {
    let oldValue = (target as any)[key]
    if (!shallow) {
      // 拿新值和老值的原始值，因为新传入的值可能是响应式数据，如果直接和 target 上原始值比较是没有意义的
      value = toRaw(value)
      oldValue = toRaw(oldValue)
      // 不是数组 && 老值是 ref && 新值不是 ref，更新 ref.value 为新值
      if (!isArray(target) && isRef(oldValue) && !isRef(value)) {
        oldValue.value = value
        return true
      }
    } else {
      // in shallow mode, objects are set as-is regardless of reactive or not
    }
    // 获取 key 值
    const hadKey =
      isArray(target) && isIntegerKey(key)
        ? Number(key) < target.length
        : hasOwn(target, key)
    // 赋值，相当于 target[key] = value
    const result = Reflect.set(target, key, value, receiver)
    // receiver 是 proxy 实例才派发更新，防止通过原型链触发拦截器触发更新
    if (target === toRaw(receiver)) {
      if (!hadKey) {
        // 如果  target 没有 key，表示新增
        trigger(target, TriggerOpTypes.ADD, key, value)
      } else if (hasChanged(value, oldValue)) {
        // 如果新旧值不相等
        trigger(target, TriggerOpTypes.SET, key, value, oldValue)
      }
    }
    return result
  }
}
```

trigger() 派发更新放到后面

有个疑问，为什么用 Reflect.get() 和 Reflect.set()，而不是直接用 target[key]？

根据 [MDN 介绍 set() 要返回一个布尔值](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Proxy/Proxy/set)，比如 Reflect.set() 会返回一个是否修改成功的布尔值，直接赋值 `target[key] = newValue`，而不返回 true 就会报错。而且不管 Proxy 怎么修改默认行为，都可以通过 Reflect 获取默认行为。get() 同理

接着是依赖收集和派发更新相关的核心内容，相关代码全部在 effect.ts 文件中，该文件主要是处理一些副作用，主要内容如下：

- 创建 effect 入口函数
- track 依赖收集
- trigger 派发更新
- cleanupEffect 清除 effect
- stop 停止 effect
- trackStack 收集栈的暂停(pauseTracking)、恢复(enableTracking)和重置(resetTracking)

我们先从入口函数看起

### effect()

源码地址：`packages/reactivity/src/effect.ts -145行`

这里主要就是暴露一个创建 `effect` 的方法

```js
export function effect<T = any>(
  fn: () => T,
  options?: ReactiveEffectOptions
): ReactiveEffectRunner {
  // 如果已经是 effect 函数，就直接拿原来的
  if ((fn as ReactiveEffectRunner).effect) {
    fn = (fn as ReactiveEffectRunner).effect.fn
  }
  // 创建 effect
  const _effect = new ReactiveEffect(fn)
  if (options) {
    extend(_effect, options)
    if (options.scope) recordEffectScope(_effect, options.scope)
  }
  // 如果 lazy 不为真就直接执行一次 effect。计算属性的 lazy 为 true
  if (!options || !options.lazy) {
    _effect.run()
  }
  // 返回
  const runner = _effect.run.bind(_effect) as ReactiveEffectRunner
  runner.effect = _effect
  return runner
}
```

可以看出主要的就是在 effect 里使用 `new ReactiveEffect` 创建了一个 `_effect` 实例，并且函数最后返回的 runner 方法就是指向 ReactiveEffect 里的 run 方法

由此可见在执行副作用函数 `effect` 方法时，实际上执行的就是这个 `run` 方法

所以我们就需要知道知道这个 `ReactiveEffect` 和它返回的 `run` 方法，里面都干了些什么

我们继续看

### ReactiveEffect

源码地址：`packages/reactivity/src/effect.ts -53行`

这里主要做的就是在依赖收集前用栈数据结构 `effectStrack` 来做 `effect` 的执行调试，保证当前 `effect` 的优先级最高，并及时清除己收集依赖的内存

需要注意的是标记完成后就会执行 `fn()` 函数，这个 fn 函数就是副作用函数封闭的函数，如果是在组件渲染，就是 fn 就是组件渲染函数，执行的时候就会就会访问数据，就会触发 `target[key]` 的 `getter`，然后触发 `track` 进行依赖收集，这也就是 Vue3 的依赖收集过程

```js
// 临时存储响应式函数
const effectStack: ReactiveEffect[] = []
// 依赖收集栈
const trackStack: boolean[] = []
// 最大嵌套深度
const maxMarkerBits = 30
export class ReactiveEffect<T = any> {
  active = true
  deps: Dep[] = []
  computed?: boolean
  allowRecurse?: boolean
  onStop?: () => void
  // dev only
  onTrack?: (event: DebuggerEvent) => void
  // dev only
  onTrigger?: (event: DebuggerEvent) => void
  constructor(
    public fn: () => T,
    public scheduler: EffectScheduler | null = null,
    scope?: EffectScope | null
  ) {
    // effectScope 相关处理，在另一个文件，这里不过多展开
    recordEffectScope(this, scope)
  }
  run() {
    if (!this.active) {
      return this.fn()
    }
    // 如果栈中没有当前的 effect
    if (!effectStack.includes(this)) {
      try {
        // activeEffect 表示当前依赖收集系统正在处理的 effect
        // 先把当前 effect 设置为全局全局激活的 effect，在 getter 中会收集 activeEffect 持有的 effect
        // 然后入栈
        effectStack.push((activeEffect = this))
        // 恢复依赖收集，因为在setup 函数自行期间，会暂停依赖收集
        enableTracking()
        // 记录递归深度位数
        trackOpBit = 1 << ++effectTrackDepth
        // 如果 effect 嵌套层数没有超过 30 层，一般超不了
        if (effectTrackDepth <= maxMarkerBits) {
          // 给依赖打标记，就是遍历 _effect 实例中的 deps 属性，给每个 dep 的 w 属性标记为 trackOpBit 的值
          initDepMarkers(this)
        } else {
          // 超过就 清除当前 effect 相关依赖 通常情况下不会
          cleanupEffect(this)
        }
        // 在执行 effect 函数，比如访问 target[key]，会触发 getter
        return this.fn()
      } finally {
        if (effectTrackDepth <= maxMarkerBits) {
          // 完成依赖标记
          finalizeDepMarkers(this)
        }
        // 恢复到上一级
        trackOpBit = 1 << --effectTrackDepth
        // 重置依赖收集状态
        resetTracking()
        // 出栈
        effectStack.pop()
        // 获取栈长度
        const n = effectStack.length
        // 将当前 activeEffect 指向栈最后一个 effect
        activeEffect = n > 0 ? effectStack[n - 1] : undefined
      }
    }
  }
  stop() {
    if (this.active) {
      cleanupEffect(this)
      if (this.onStop) {
        this.onStop()
      }
      this.active = false
    }
  }
}
```

### track()

源码地址：`packages/reactivity/src/effect.ts -188行`

`track` 就是依赖收集器，负责把依赖收集起来统一放到一个依赖管理中心

```js
// targetMap 为依赖管理中心，用于存储响应式函数、目标对象、键之间的映射关系
// 相当于这样
// targetMap(weakmap)={
//    target1(map):{
//      key1(dep):[effect1,effect2]
//      key2(dep):[effect1,effect2]
//    }
// }
// 给每个 target 创建一个 map，每个 key 对应着一个 dep
// 用 dep 来收集依赖函数，监听 key 值变化，触发 dep 中的依赖函数
const targetMap = new WeakMap<any, KeyToDepMap>()
export function isTracking() {
  return shouldTrack && activeEffect !== undefined
}
export function track(target: object, type: TrackOpTypes, key: unknown) {
  // 如果当前没有激活 effect，就不用收集
  if (!isTracking()) {
    return
  }
  // 从依赖管理中心里获取 target
  let depsMap = targetMap.get(target)
  if (!depsMap) {
    // 如果没有就创建一个
    targetMap.set(target, (depsMap = new Map()))
  }
  // 获取 key 对应的 dep 集合
  let dep = depsMap.get(key)
  if (!dep) {
    // 没有就创建
    depsMap.set(key, (dep = createDep()))
  }
  // 开发环境和非开发环境
  const eventInfo = __DEV__
    ? { effect: activeEffect, target, type, key }
    : undefined
  trackEffects(dep, eventInfo)
}
```

### trackEffects()

源码地址：`packages/reactivity/src/effect.ts -212行`

这里把当前激活的 effect 收集进对应的 effect 集合，也就是 `dep`

这里了解一下两个标识符

`dep.n`：n 是 `newTracked` 的缩写，表示是否是最新收集的(是否当前层)  
`dep.w`：w 是 `wasTracked` 的缩写，表示是否已经被收集，避免重复收集

```js
export function trackEffects(
  dep: Dep,
  debuggerEventExtraInfo?: DebuggerEventExtraInfo
) {
  let shouldTrack = false
  // 如果 effect 嵌套层数没有超过 30 层，上面说过了
  if (effectTrackDepth <= maxMarkerBits) {
    if (!newTracked(dep)) {
      // 标记新依赖
      dep.n |= trackOpBit
      // 已经被收集的依赖不需要重复收集
      shouldTrack = !wasTracked(dep)
    }
  } else {
    // 超过了 就切换清除依赖模式
    shouldTrack = !dep.has(activeEffect!)
  }
  // 如果可以收集
  if (shouldTrack) {
    // 收集当前激活的 effect 作为依赖
    dep.add(activeEffect!)
    // 当前激活的 effect 收集 dep 集合
    activeEffect!.deps.push(dep)
    // 开发环境下触发 onTrack 事件
    if (__DEV__ && activeEffect!.onTrack) {
      activeEffect!.onTrack(
        Object.assign(
          {
            effect: activeEffect!
          },
          debuggerEventExtraInfo
        )
      )
    }
  }
}
```

### trigger()

源码地址：`packages/reactivity/src/effect.ts -243行`

`trigger` 是 track 收集的依赖对应的触发器，也就是负责根据映射关系，获取响应式函数，再派发通知 `triggerEffects` 进行更新

```js
export function trigger(
  target: object,
  type: TriggerOpTypes,
  key?: unknown,
  newValue?: unknown,
  oldValue?: unknown,
  oldTarget?: Map<unknown, unknown> | Set<unknown>
) {
  // 从依赖管理中心中获取依赖
  const depsMap = targetMap.get(target)
  // 没有被收集过的依赖，直接返回
  if (!depsMap) {
    return
  }

  let deps: (Dep | undefined)[] = []
  // 触发trigger 的时候传进来的类型是清除类型
  if (type === TriggerOpTypes.CLEAR) {
    // 往队列中添加关联的所有依赖，准备清除
    deps = [...depsMap.values()]
  } else if (key === 'length' && isArray(target)) {
    // 如果是数组类型的，并且是数组的 length 改变时
    depsMap.forEach((dep, key) => {
      // 如果数组长度变短时，需要做已删除数组元素的 effects 和 trigger
      // 也就是索引号 >= 数组最新的length的元素们对应的 effects，要将它们添加进队列准备清除
      if (key === 'length' || key >= (newValue as number)) {
        deps.push(dep)
      }
    })
  } else {
    // 如果 key 不是 undefined，就添加对应依赖到队列，比如新增、修改、删除
    if (key !== void 0) {
      deps.push(depsMap.get(key))
    }
    // 新增、修改、删除分别处理
    switch (type) {
      case TriggerOpTypes.ADD: // 新增
        ...
        break
      case TriggerOpTypes.DELETE: // 删除
        ...
        break
      case TriggerOpTypes.SET: // 修改
        ...
        break
    }
  }
  // 到这里就拿到了 targetMap[target][key]，并存到 deps 里
  // 接着是要将对应的 effect 取出，调用 triggerEffects 执行

  // 判断开发环境，传入eventInfo
  const eventInfo = __DEV__
    ? { target, type, key, newValue, oldValue, oldTarget }
    : undefined

  if (deps.length === 1) {
    if (deps[0]) {
      if (__DEV__) {
        triggerEffects(deps[0], eventInfo)
      } else {
        triggerEffects(deps[0])
      }
    }
  } else {
    const effects: ReactiveEffect[] = []
    for (const dep of deps) {
      if (dep) {
        effects.push(...dep)
      }
    }
    if (__DEV__) {
      triggerEffects(createDep(effects), eventInfo)
    } else {
      triggerEffects(createDep(effects))
    }
  }
}
```

### triggerEffects()

源码地址：`packages/reactivity/src/effect.ts -330行`

执行 effect 函数，也就是『派发更新』中的更新了

```js
export function triggerEffects(
  dep: Dep | ReactiveEffect[],
  debuggerEventExtraInfo?: DebuggerEventExtraInfo
) {
  // 遍历 effect 的集合函数
  for (const effect of isArray(dep) ? dep : [...dep]) {
    /** 
      这里判断 effect !== activeEffect的原因是：不能和当前effect 相同
      比如：count.value++，如果这是个effect，会触发getter，track收集了当前激活的 effect，
      然后count.value = count.value+1 会触发setter，执行trigger，
      就会陷入一个死循环，所以要过滤当前的 effect
    */
    if (effect !== activeEffect || effect.allowRecurse) {
      if (__DEV__ && effect.onTrigger) {
        effect.onTrigger(extend({ effect }, debuggerEventExtraInfo))
      }
      // 如果 scheduler 就执行，计算属性有 scheduler
      if (effect.scheduler) {
        effect.scheduler()
      } else {
        // 执行 effect 函数
        effect.run()
      }
    }
  }
}
```

### 创建 ref

源码地址：`packages/reactivity/src/ref.ts`

这里开始主要就是处理 `ref` 相关的了，先看一下几个相关函数，后面会用的

```js
// 判断是不是 ref
export function isRef(r: any): r is Ref {
  return Boolean(r && r.__v_isRef === true)
}
// 创建 ref
function createRef(rawValue: unknown, shallow: boolean) {
  if (isRef(rawValue)) { // 如果已经是 ref 就直接返回
    return rawValue
  }
  // 调用 RefImpl 创建并返回 ref
  return new RefImpl(rawValue, shallow)
}
// 创建一个浅层 ref
export function shallowRef(value?: unknown) {
  return createRef(value, true)
}
// 卸载一个 ref
export function unref<T>(ref: T | Ref<T>): T {
  return isRef(ref) ? (ref.value as any) : ref
}

```

### RefImpl

源码地址：`packages/reactivity/src/ref.ts -87行`

从上面我们知道 `ref` 对象是通过 `new RefImpl()` 创建的，RefImpl 类的实现比较简单，这里就不多废话了，请看注释

```js
class RefImpl<T> {
  private _value: T
  private _rawValue: T

  public dep?: Dep = undefined
  // 每一个 ref 实例下都有一个__v_isRef 的只读属性，标识它是一个 ref
  public readonly __v_isRef = true

  constructor(value: T, public readonly _shallow: boolean) {
    // 判断是不是浅比较，如果不是就拿老值
    this._rawValue = _shallow ? value : toRaw(value)
    // 判断是不是浅比较，如果不是就调convert，判断如果是对象就调用 reactive()
    this._value = _shallow ? value : convert(value)
  }
  // ref.value 这样取值
  get value() {
    // 进行依赖收集
    trackRefValue(this)
    return this._value
  }
  set value(newVal) {
    // 如果是浅比较，就取新值，不是就取老值
    newVal = this._shallow ? newVal : toRaw(newVal)
    // 比较新旧值
    if (hasChanged(newVal, this._rawValue)) {
      // 值已更换，重新赋值
      this._rawValue = newVal
      this._value = this._shallow ? newVal : convert(newVal)
      // 派发更新
      triggerRefValue(this, newVal)
    }
  }
}
```

### trackRefValue()

源码地址：`packages/reactivity/src/ref.ts -29行`

这里主要做一些 ref 依赖收集之前的工作，主要就是判断是否激活了 effect，有没有收集过依赖的 effect，没有就创建一个 dep，准备收集，然后调用本文上面的 `trackEffects` 进行依赖收集

```js
export function trackRefValue(ref: RefBase<any>) {
  // 如果激活了 effect，就收集
  if (isTracking()) {
    ref = toRaw(ref)
    // 如果该属性没有没有收集过依赖函数，就创建一个 dep，用来存放依赖 effect
    if (!ref.dep) {
      ref.dep = createDep()
    }
    // 开发环境
    if (__DEV__) {
      trackEffects(ref.dep, {
        target: ref,
        type: TrackOpTypes.GET,
        key: "value",
      })
    } else {
      // 调用本文上面的 trackEffects 收集依赖
      trackEffects(ref.dep)
    }
  }
}
```

### triggerRefValue()

源码地址：`packages/reactivity/src/ref.ts -47行`

这里进行 ref 派发更新，源码比较简单，没啥说的，就是区分一下开发环境，然后执行本文上面的 `triggerEffects` 执行对应在的 effect 函数进行更新

```js
export function triggerRefValue(ref: RefBase<any>, newVal?: any) {
  ref = toRaw(ref)
  if (ref.dep) {
    if (__DEV__) {
      triggerEffects(ref.dep, {
        target: ref,
        type: TriggerOpTypes.SET,
        key: "value",
        newValue: newVal,
      })
    } else {
      triggerEffects(ref.dep)
    }
  }
}
```

到这里，Vue3 的响应式对象的源码就基本上剖析结束了
