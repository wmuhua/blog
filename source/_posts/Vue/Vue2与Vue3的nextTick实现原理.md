---
title: Vue2与Vue3的nextTick实现原理
date: 2020-09-12 18:10:11
author: 沐华
img: https://cdn.jsdelivr.net/gh/wmuhua/cdn@main/blog/23.jpg
top: false
cover: true
coverImg: /images/1.jpg
password:
toc: false
mathjax: false
summary: 都会用 nextTick，那么它到底是怎么实现的呢，在 Vue2 和 Vue3 中又有什么区别呢？
categories: Vue
tags:
  - Vue
  - nextTick
  - Vue3
  - 源码
---

都会用 nextTick，也都知道 nextTick 作用是在下次 DOM 更新循环结束之后，执行延迟回调，就可以拿到更新后的 DOM 相关信息

那么它到底是怎么实现的呢，在 Vue2 和 Vue3 中又有什么区别呢？本文将结合案例介绍执行原理再深入源码，全部注释，包你一看就会

在进入 nextTick 实现原理之前先稍微回顾一下 JS 的执行机制，因为这与 nextTick 的实现息息相关

## JS 执行机制

我们都知道 JS 是单线程的，一次只能干一件事，即**同步**，就是说所有的任务都需要排队，后面的任务需要等前面的任务执行完才能执行，如果前面的任务耗时过长，后面的任务就需要一直等，这是非常影响用户体验的，所以才出现了**异步**的概念

`同步任务`：指排队在主线程上依次执行的任务  
`异步任务`：不进入主线程，而进入任务队列的任务，又分为宏任务和微任务  
`宏任务`： 渲染事件、请求、script、setTimeout、setInterval、Node 中的 setImmediate 等  
`微任务`： Promise.then、MutationObserver(监听 DOM)、Node 中的 Process.nextTick 等

当执行栈中的同步任务执行完后，就会去任务队列中拿一个宏任务放到执行栈中执行，执行完该宏任务中的所有微任务，再到任务队列中拿宏任务，即一个宏任务、所有微任务、渲染、一个宏任务、所有微任务、渲染...(不是所有微任务之后都会执行渲染)，如此形成循环，即`事件循环(EventLoop)`

`nextTick` 就是创建一个异步任务，那么它自然要等到同步任务执行完成后才执行

我们先结合例子弄懂执行原理，再深入源码

## Vue2

### nextTick 用法

看例子，比如当 DOM 内容改变后，我们需要获取最新的高度

```js
<template>
  <div>{{ name }}</div>
</template>
<script>
export default {
  data() {
    return {
      name: ""
    }
  },
  mounted() {
    console.log(this.$el.clientHeight) // 0
    this.name = "沐华"
    console.log(this.$el.clientHeight) // 0
    this.$nextTick(() => {
      console.log(this.$el.clientHeight) // 18
    });
  }
};
</script>
```

为什么在 nextTick 里就能拿到最新的 DOM 相关信息？是怎么拿到的，我们来分析一下原理

### 原理分析

在执行 `this.name = '沐华'` 的时候，就会触发 `Watcher` 更新，watcher 会把自己放到一个队列

> 用队列的原因是比如多个数据变更就更新视图多次的话，性能上就不好了，所以对视图更新做一个异步更新的队列，避免重复计算和不必要的 DOM 操作，在下一轮事件循环的时候刷新队列，并执行已去重的任务(nextTick 的回调函数)，更新视图

然后调用 `nextTick()`，响应式派发更新的源码在这一块是这样的，地址：`src/core/observer/scheduler.js - 164行`

```js
export function queueWatcher (watcher: Watcher) {
  ...
  // 因为每次派发更新都会引起渲染，所以把所有 watcher 都放到 nextTick 里调用
  nextTick(flushSchedulerQueue)
}
```

这里参数 `flushSchedulerQueue` 方法就会被放入事件循环，主线程任务的行完后就会执行这个函数，对 watcher 队列排序、遍历、执行 watcher 对应的 run 方法，然后 render，更新视图

也就是说 `this.name = '沐华'` 的时候，任务队列可以简单理解成这样 `[flushSchedulerQueue]`

然后下一行 `console.log(...)`，由于会更新视图的任务 `flushSchedulerQueue` 在任务队列里没有执行，所以无法拿到更新后的视图

然后执行到 `this.$nextTick(fn)` 的时候，添加一个异步任务，这时的任务队列可以简单理解成这样 `[flushSchedulerQueue, fn]`

然后同步任务就执行完了，接着按顺序执行任务队列里的任务，第一个任务执行就会更新视图，后面自然能得到更新后的视图了

### nextTick 源码剖析

源码版本：`2.6.14`，源码地址：`src/core/util/next-tick.js`

这里整个源码分为两部分，一是判断当前环境能使用的最合适的 `API` 并保存异步函数，二是调用异步函数 执行回调队列

#### 环境判断

主要是判断用哪个宏任务或微任务，因为宏任务耗费的时间是大于微任务的，所以成先使用微任务，判断顺序如下

- `Promise`
- `MutationObserver`
- `setImmediate`
- `setTimeout`

```js
export let isUsingMicroTask = false // 是否启用微任务开关
const callbacks = [] // 回调队列
let pending = false // 异步控制开关，标记是否正在执行回调函数

// 该方法负责执行队列中的全部回调
function flushCallbacks() {
  // 重置异步开关
  pending = false
  // 防止nextTick里有nextTick出现的问题
  // 所以执行之前先备份并清空回调队列
  const copies = callbacks.slice(0)
  callbacks.length = 0
  // 执行任务队列
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}
let timerFunc // 用来保存调用异步任务方法
// 判断当前环境是否支持原生 Promise
if (typeof Promise !== "undefined" && isNative(Promise)) {
  // 保存一个异步任务
  const p = Promise.resolve()
  timerFunc = () => {
    // 执行回调函数
    p.then(flushCallbacks)
    // ios 中可能会出现一个回调被推入微任务队列，但是队列没有刷新的情况
    // 所以用一个空的计时器来强制刷新任务队列
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
} else if (
  !isIE &&
  typeof MutationObserver !== "undefined" &&
  (isNative(MutationObserver) ||
    MutationObserver.toString() === "[object MutationObserverConstructor]")
) {
  // 不支持 Promise 的话，在支持MutationObserver的非 IE 环境下
  // 如 PhantomJS, iOS7, Android 4.4
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true,
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
} else if (typeof setImmediate !== "undefined" && isNative(setImmediate)) {
  // 使用setImmediate，虽然也是宏任务，但是比setTimeout更好
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  // 以上都不支持的情况下，使用 setTimeout
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}
```

环境判断结束就会得到一个延迟回调函数 `timerFunc`

然后进入核心的 nextTick

#### nextTick()

我们用 `Vue.nextTick()` 或者 `this.$nextTick()` 都是调用 `nextTick()` 这个方法

这里代码不多，主要逻辑就是：

- 把传入的回调函数放进回调队列 `callbacks`
- 执行保存的异步任务 `timeFunc`，就会遍历 `callbacks` 执行相应的回调函数了

```js
export function nextTick(cb?: Function, ctx?: Object) {
  let _resolve
  // 把回调函数放入回调队列
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, "nextTick")
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    // 如果异步开关是开的，就关上，表示正在执行回调函数，然后执行回调函数
    pending = true
    timerFunc()
  }
  // 如果没有提供回调，并且支持 Promise，就返回一个 Promise
  if (!cb && typeof Promise !== "undefined") {
    return new Promise((resolve) => {
      _resolve = resolve
    })
  }
}
```

可以看到最后有返回一个 `Promise` 是可以让我们在不传参的时候用的，如下

```js
this.$nextTick().then(()=>{ ... })
```

## Vue3

### nextTick 用法

先看个例子，点击按钮更新 DOM 内容，并获取最新的 DOM 内容

```js
 <template>
     <div ref="test">{{name}}</div>
     <el-button @click="handleClick">按钮</el-button>
 </template>
 <script setup>
     import { ref, nextTick } from 'vue'
     const name = ref("沐华")
     const test = ref(null)
     async function handleClick(){
         name.value = '掘金'
         console.log(test.value.innerText) // 沐华
         await nextTick()
         console.log(test.value.innerText) // 掘金
     }
     return { name, test, handleClick }
 </script>
```

Vue3 里这一块有大改，不过事件循环的原理还是一样，只是加了几个专门维护队列的方法，以及关联到 `effect`，不过好在这里源码的代码不多，所以不如直接看源码会更容易理解

### nextTick 源码剖析

源码版本：`3.2.11`，源码地址：`packages/runtime-core/src/sheduler.ts`

```js
const resolvedPromise: Promise<any> = Promise.resolve()
let currentFlushPromise: Promise<void> | null = null

export function nextTick<T = void>(
  this: T,
  fn?: (this: T) => void
): Promise<void> {
  const p = currentFlushPromise || resolvedPromise
  return fn ? p.then(this ? fn.bind(this) : fn) : p
}
```

就一个 Promise，没了

就这！！！

<div align="center"><img style="display:inline-block" src="https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f10c2a388e8b45479d2c7b718f2bd161~tplv-k3u1fbpfcp-watermark.image" width="300" /></div>

好吧，认真点

可以看出 nextTick 接受一个函数为参数，同时会创建一个微任务

在我们页面调用 `nextTick` 的时候，会执行该函数，把我们的参数 `fn` 赋值给 `p.then(fn)`，在队列的任务完成后，fn 就执行了

由于加了几个维护队列的方法，所以执行顺序是这样的：

`queueJob` -> `queueFlush` -> `flushJobs` -> `nextTick参数的 fn`

现在不知道都是干嘛的不要紧，几分钟后你就会清楚了

我们按顺序来，先看一下入口函数 `queueJob` 是在哪里调用的，看代码

```js
// packages/runtime-core/src/renderer.ts - 1555行
function baseCreateRenderer(){
  const setupRenderEffect: SetupRenderEffectFn = (...) => {
    const effect = new ReactiveEffect(
      componentUpdateFn,
      () => queueJob(instance.update), // 当作参数传入
      instance.scope
    )
  }
}
```

在 `ReactiveEffect` 这边接收过来的形参就是 `scheduler`，最终被用到了下面这里，看过响应式源码的这里就熟悉了，就是派发更新的地方

```js
// packages/reactivity/src/effect.ts - 330行
export function triggerEffects(
  ...
  if (effect.scheduler) {
    effect.scheduler()
  } else {
    effect.run()
  }
}
```

然后是 `queueJob` 里面干了什么？我们一个一个的来

#### queueJob()

该方法负责维护主任务队列，接受一个函数作为参数，为待入队任务，会将参数 `push` 到 `queue` 队列中，有唯一性判断。会在当前宏任务执行结束后，清空队列

```js
const queue: SchedulerJob[] = []

export function queueJob(job: SchedulerJob) {
  // 主任务队列为空 或者 有正在执行的任务且没有在主任务队列中  && job 不能和当前正在执行任务及后面待执行任务相同
  if (
    (!queue.length ||
      !queue.includes(
        job,
        isFlushing && job.allowRecurse ? flushIndex + 1 : flushIndex
      )) &&
    job !== currentPreFlushParentJob
  ) {
    // 可以入队就添加到主任务队列
    if (job.id == null) {
      queue.push(job)
    } else {
      // 否则就删除
      queue.splice(findInsertionIndex(job.id), 0, job)
    }
    // 创建微任务
    queueFlush()
  }
}
```

#### queueFlush()

该方法负责尝试创建微任务，等待任务队列执行

```js
let isFlushing = false // 是否正在执行
let isFlushPending = false // 是否正在等待执行
const resolvedPromise: Promise<any> = Promise.resolve() // 微任务创建器
let currentFlushPromise: Promise<void> | null = null // 当前任务

function queueFlush() {
  // 当前没有微任务
  if (!isFlushing && !isFlushPending) {
    // 避免在事件循环周期内多次创建新的微任务
    isFlushPending = true
    // 创建微任务，把 flushJobs 推入任务队列等待执行
    currentFlushPromise = resolvedPromise.then(flushJobs)
  }
}
```

#### flushJobs()

该方法负责处理队列任务，主要逻辑如下：

- 先处理前置任务队列
- 根据 `Id` 排队队列
- 遍历执行队列任务
- 执行完毕后清空并重置队列
- 执行后置队列任务
- 如果还有就递归继续执行

```js
function flushJobs(seen?: CountMap) {
  isFlushPending = false // 是否正在等待执行
  isFlushing = true // 正在执行
  if (__DEV__) seen = seen || new Map() // 开发环境下
  flushPreFlushCbs(seen) // 执行前置任务队列
  // 根据 id 排序队列，以确保
  // 1. 从父到子，因为父级总是在子级前面先创建
  // 2. 如果父组件更新期间卸载了组件，就可以跳过
  queue.sort((a, b) => getId(a) - getId(b))
  try {
    // 遍历主任务队列，批量执行更新任务
    for (flushIndex = 0; flushIndex < queue.length; flushIndex++) {
      const job = queue[flushIndex]
      if (job && job.active !== false) {
        if (__DEV__ && checkRecursiveUpdates(seen!, job)) {
          continue
        }
        callWithErrorHandling(job, null, ErrorCodes.SCHEDULER)
      }
    }
  } finally {
    flushIndex = 0 // 队列任务执行完，重置队列索引
    queue.length = 0 // 清空队列
    flushPostFlushCbs(seen) // 执行后置队列任务
    isFlushing = false  // 重置队列执行状态
    currentFlushPromise = null // 重置当前微任务为 Null
    // 如果主任务队列、前置和后置任务队列还有没被清空，就继续递归执行
    if ( queue.length || pendingPreFlushCbs.length || pendingPostFlushCbs.length ) {
      flushJobs(seen)
    }
  }
}
```

#### flushPreFlushCbs()

该方法负责执行前置任务队列，说明都写在注释里了

```js
export function flushPreFlushCbs( seen?: CountMap, parentJob: SchedulerJob | null = null) {
  // 如果待处理的队列不为空
  if (pendingPreFlushCbs.length) {
    currentPreFlushParentJob = parentJob
    // 保存队列中去重后的任务为当前活动的队列
    activePreFlushCbs = [...new Set(pendingPreFlushCbs)]
    // 清空队列
    pendingPreFlushCbs.length = 0
    // 开发环境下
    if (__DEV__) { seen = seen || new Map() }
    // 遍历执行队列里的任务
    for ( preFlushIndex = 0; preFlushIndex < activePreFlushCbs.length; preFlushIndex+ ) {
      // 开发环境下
      if ( __DEV__ && checkRecursiveUpdates(seen!, activePreFlushCbs[preFlushIndex])) {
        continue
      }
      activePreFlushCbs[preFlushIndex]()
    }
    // 清空当前活动的任务队列
    activePreFlushCbs = null
    preFlushIndex = 0
    currentPreFlushParentJob = null
    // 递归执行，直到清空前置任务队列，再往下执行异步更新队列任务
    flushPreFlushCbs(seen, parentJob)
  }
}
```

#### flushPostFlushCbs()

该方法负责执行后置任务队列，说明都写在注释里了

```js
let activePostFlushCbs: SchedulerJob[] | null = null

export function flushPostFlushCbs(seen?: CountMap) {
  // 如果待处理的队列不为空
  if (pendingPostFlushCbs.length) {
    // 保存队列中去重后的任务
    const deduped = [...new Set(pendingPostFlushCbs)]
    // 清空队列
    pendingPostFlushCbs.length = 0
    // 如果当前已经有活动的队列，就添加到执行队列的末尾，并返回
    if (activePostFlushCbs) {
      activePostFlushCbs.push(...deduped)
      return
    }
    // 赋值为当前活动队列
    activePostFlushCbs = deduped
    // 开发环境下
    if (__DEV__) seen = seen || new Map()
    // 排队队列
    activePostFlushCbs.sort((a, b) => getId(a) - getId(b))
    // 遍历执行队列里的任务
    for ( postFlushIndex = 0; postFlushIndex < activePostFlushCbs.length; postFlushIndex++ ) {
      if ( __DEV__ && checkRecursiveUpdates(seen!, activePostFlushCbs[postFlushIndex])) {
        continue
      }
      activePostFlushCbs[postFlushIndex]()
    }
    // 清空当前活动的任务队列
    activePostFlushCbs = null
    postFlushIndex = 0
  }
}
```

整个 nextTick 的源码到这就解析完啦
