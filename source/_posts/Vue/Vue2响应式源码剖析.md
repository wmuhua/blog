---
title: Vue2响应式源码剖析
date: 2020-06-12 22:39:06
author: 沐华
img: https://cdn.jsdelivr.net/gh/wmuhua/cdn@main/blog/22.jpg
top: false
cover: true
coverImg: /images/1.jpg
password:
toc: false
mathjax: false
summary: 响应式这种熟悉的问题还有不会的吗？
categories: Vue
tags:
  - Vue
  - 响应式
  - 源码
---

先看张图，了解一下大体流程和要做的事

![响应式原理流程.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0938e472962745328bf35297f55216bb~tplv-k3u1fbpfcp-watermark.image?)

## 初始化

在 new Vue 初始化的时候，会对我们组件的数据 props 和 data 进行初始化，由于本文主要就是介绍响应式，所以其他的不做过多说明来，看一下源码

源码地址：`src/core/instance/init.js - 15行`

```js
export function initMixin (Vue: Class<Component>) {
  // 在原型上添加 _init 方法
  Vue.prototype._init = function (options?: Object) {
    ...
    vm._self = vm
    initLifecycle(vm) // 初始化实例的属性、数据：$parent, $children, $refs, $root, _watcher...等
    initEvents(vm) // 初始化事件：$on, $off, $emit, $once
    initRender(vm) // 初始化渲染： render, mixin
    callHook(vm, 'beforeCreate') // 调用生命周期钩子函数
    initInjections(vm) // 初始化 inject
    initState(vm) // 初始化组件数据：props, data, methods, watch, computed
    initProvide(vm) // 初始化 provide
    callHook(vm, 'created') // 调用生命周期钩子函数
    ...
  }
}
```

初始化这里调用了很多方法，每个方法都做着不同的事，而关于响应式主要就是组件内的数据 `props`、`data`。这一块的内容就是在 `initState()` 这个方法里，所以我们进入这个方法源码看一下

### initState()

源码地址：`src/core/instance/state.js - 49行`

```js
export function initState(vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  // 初始化 props
  if (opts.props) initProps(vm, opts.props)
  // 初始化 methods
  if (opts.methods) initMethods(vm, opts.methods)
  // 初始化 data
  if (opts.data) {
    initData(vm)
  } else {
    // 没有 data 的话就默认赋值为空对象，并监听
    observe((vm._data = {}), true /* asRootData */)
  }
  // 初始化 computed
  if (opts.computed) initComputed(vm, opts.computed)
  // 初始化 watch
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```

又是调用一堆初始化的方法，我们还是直奔主题，取我们响应式数据相关的，也就是 `initProps()`、`initData()`、`observe()`

一个一个继续扒，非得整明白响应式的全部过程

### initProps()

源码地址：`src/core/instance/state.js - 65行`

这里主要做的是：

- 遍历父组件传进来的 `props` 列表
- 校验每个属性的命名、类型、default 属性等，都没有问题就调用 `defineReactive` 设置成响应式
- 然后用 `proxy()` 把属性代理到当前实例上，如把 `vm._props.xx` 变成 `vm.xx`，就可以访问

```js
function initProps(vm: Component, propsOptions: Object) {
  // 父组件传入子组件的 props
  const propsData = vm.$options.propsData || {}
  // 经过转换后最终的 props
  const props = (vm._props = {})
  // 存放 props 的 key，就算 props 值空了，key 也会在里面
  const keys = (vm.$options._propKeys = [])
  const isRoot = !vm.$parent
  // 转换非根实例的 props
  if (!isRoot) {
    toggleObserving(false)
  }
  for (const key in propsOptions) {
    keys.push(key)
    // 校验 props 类型、default 属性等
    const value = validateProp(key, propsOptions, propsData, vm)
    // 在非生产环境中
    if (process.env.NODE_ENV !== "production") {
      const hyphenatedKey = hyphenate(key)
      if (
        isReservedAttribute(hyphenatedKey) ||
        config.isReservedAttr(hyphenatedKey)
      ) {
        warn(`hyphenatedKey 是保留属性，不能用作组件 prop`)
      }
      // 把 props 设置成响应式的
      defineReactive(props, key, value, () => {
        // 如果用户修改 props 发出警告
        if (!isRoot && !isUpdatingChildComponent) {
          warn(`避免直接改变 prop`)
        }
      })
    } else {
      // 把 props 设置为响应式
      defineReactive(props, key, value)
    }
    // 把不在默认 vm 上的属性，代理到实例上
    // 可以让 vm._props.xx 通过 vm.xx 访问
    if (!(key in vm)) {
      proxy(vm, `_props`, key)
    }
  }
  toggleObserving(true)
}
```

### initData()

源码地址：`src/core/instance/state.js - 113行`

这里主要做的是：

- 初始化一个 data，并拿到 keys 集合
- 遍历 keys 集合，来判断有没有和 props 里的属性名或者 methods 里的方法名重名的
- 没有问题就通过 `proxy()` 把 data 里的每一个属性都代理到当前实例上，就可以通过 `this.xx` 访问了
- 最后再调用 `observe` 监听整个 data

```js
function initData(vm: Component) {
  // 获取当前实例的 data
  let data = vm.$options.data
  // 判断 data 的类型
  data = vm._data = typeof data === "function" ? getData(data, vm) : data || {}
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== "production" && warn(`数据函数应该返回一个对象`)
  }
  // 获取当前实例的 data 属性名集合
  const keys = Object.keys(data)
  // 获取当前实例的 props
  const props = vm.$options.props
  // 获取当前实例的 methods 对象
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    // 非生产环境下判断 methods 里的方法是否存在于 props 中
    if (process.env.NODE_ENV !== "production") {
      if (methods && hasOwn(methods, key)) {
        warn(`Method 方法不能重复声明`)
      }
    }
    // 非生产环境下判断 data 里的属性是否存在于 props 中
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== "production" && warn(`属性不能重复声明`)
    } else if (!isReserved(key)) {
      // 都不重名的情况下，代理到 vm 上
      // 可以让 vm._data.xx 通过 vm.xx 访问
      proxy(vm, `_data`, key)
    }
  }
  // 监听 data
  observe(data, true /* asRootData */)
}
```

### observe()

源码地址：`src/core/observer/index.js - 110行`

这个方法主要就是用来给数据加上监听器的

这里主要做的是：

- 如果是 vnode 的对象类型或者不是引用类型，就直接跳出
- 否则就给没有添加 Observer 的数据添加一个 Observer，也就是监听者

```js
export function observe(value: any, asRootData: ?boolean): Observer | void {
  // 如果不是'object'类型 或者是 vnode 的对象类型就直接返回
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  // 使用缓存的对象
  if (hasOwn(value, "__ob__") && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    // 创建监听者
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```

### Observer

源码地址：`src/core/observer/index.js - 37行`

这是一个类，作用是把一个正常的数据成可观测的数据

这里主要做的是：

- 给当前 value 打上已经是响应式属性的标记，避免重复操作
- 然后判断数据类型
  - 如果是对象，就遍历对象，调用 defineReactive()创建响应式对象
  - 如果是数组，就遍历数组，调用 observe()对每一个元素进行监听

```js
export class Observer {
  value: any
  dep: Dep
  vmCount: number // 根对象上的 vm 数量
  constructor(value: any) {
    this.value = value
    this.dep = new Dep()
    this.vmCount = 0
    // 给 value 添加 __ob__ 属性，值为value 的 Observe 实例
    // 表示已经变成响应式了，目的是对象遍历时就直接跳过，避免重复操作
    def(value, "__ob__", this)
    // 类型判断
    if (Array.isArray(value)) {
      // 判断数组是否有__proty__
      if (hasProto) {
        // 如果有就重写数组的方法
        protoAugment(value, arrayMethods)
      } else {
        // 没有就通过 def，也就是Object.defineProperty 去定义属性值
        copyAugment(value, arrayMethods, arrayKeys)
      }
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }
  // 如果是对象类型
  walk(obj: Object) {
    const keys = Object.keys(obj)
    // 遍历对象所有属性，转为响应式对象，也是动态添加 getter 和 setter，实现双向绑定
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }
  // 监听数组
  observeArray(items: Array<any>) {
    // 遍历数组，对每一个元素进行监听
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```

### defineReactive()

源码地址：`src/core/observer/index.js - 135行`

这个方法的作用是定义响应式对象

这里主要做的是：

- 先初始化一个 dep 实例
- 如果是对象就调用 observe，递归监听，以保证不管结构嵌套多深，都能变成响应式对象
- 然后调用 Object.defineProperty() 劫持对象属性的 getter 和 getter
- 如果获取时，触发 getter 会调用 dep.depend() 把观察者 push 到依赖的数组 subs 里去，也就是依赖收集
- 如果更新时，触发 setter 会做以下操作
  - 新值没有变化或者没有 setter 属性的直接跳出
  - 如果新值是对象就调用 observe() 递归监听
  - 然后调用 dep.notify() 派发更新

```js
export function defineReactive(
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  // 创建 dep 实例
  const dep = new Dep()
  // 拿到对象的属性描述符
  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }
  // 获取自定义的 getter 和 setter
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }
  // 如果 val 是对象的话就递归监听
  // 递归调用 observe 就可以保证不管对象结构嵌套有多深，都能变成响应式对象
  let childOb = !shallow && observe(val)
  // 截持对象属性的 getter 和 setter
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    // 拦截 getter，当取值时会触发该函数
    get: function reactiveGetter() {
      const value = getter ? getter.call(obj) : val
      // 进行依赖收集
      // 初始化渲染 watcher 时访问到需要双向绑定的对象，从而触发 get 函数
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    // 拦截 setter，当值改变时会触发该函数
    set: function reactiveSetter(newVal) {
      const value = getter ? getter.call(obj) : val
      // 判断是否发生变化
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      if (process.env.NODE_ENV !== "production" && customSetter) {
        customSetter()
      }
      // 没有 setter 的访问器属性
      if (getter && !setter) return
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      // 如果新值是对象的话递归监听
      childOb = !shallow && observe(newVal)
      // 派发更新
      dep.notify()
    },
  })
}
```

上面说了通过 dep.depend 来做依赖收集，可以说 Dep 就是整个 getter 依赖收集的核心了

## 依赖收集

依赖收集的核心是 Dep，而且它与 Watcher 也是密不可分的，我们来看一下

### Dep

源码地址：`src/core/observer/dep.js`

这是一个类，它实际上就是对 Watcher 的一种管理

这里首先初始化一个 subs 数组，用来存放依赖，也就是观察者，谁依赖这个数据，谁就在这个数组里，然后定义几个方法来对依赖添加、删除、通知更新等

另外它有一个静态属性 target，这是一个全局的 Watcher，也表示同一时间只能存在一个全局的 Watcher

```js
let uid = 0
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;
  constructor () {
    this.id = uid++
    this.subs = []
  }
  // 添加观察者
  addSub (sub: Watcher) {
    this.subs.push(sub)
  }
  // 移除观察者
  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }
  depend () {
    if (Dep.target) {
      // 调用 Watcher 的 addDep 函数
      Dep.target.addDep(this)
    }
  }
  // 派发更新(下一章节介绍)
  notify () {
    ...
  }
}
// 同一时间只有一个观察者使用，赋值观察者
Dep.target = null
const targetStack = []

export function pushTarget (target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}

export function popTarget () {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}
```

### Watcher

源码地址：`src/core/observer/watcher.js`

Watcher 也是一个类，也叫观察者(订阅者)，这里干的活还挺复杂的，而且还串连了渲染和编译

先看源码吧，再来捋一下整个依赖收集的过程

```js
let uid = 0
export default class Watcher {
  ...
  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    this.vm = vm
    if (isRenderWatcher) {
      vm._watcher = this
    }
    vm._watchers.push(this)
    // Watcher 实例持有的 Dep 实例的数组
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    this.value = this.lazy
      ? undefined
      : this.get()
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
    }
  }
  get ()
    // 该函数用于缓存 Watcher
    // 因为在组件含有嵌套组件的情况下，需要恢复父组件的 Watcher
    pushTarget(this)
    let value
    const vm = this.vm
    try {
      // 调用回调函数，也就是upcateComponent，对需要双向绑定的对象求值，从而触发依赖收集
      value = this.getter.call(vm, vm)
    } catch (e) {
      ...
    } finally {
      // 深度监听
      if (this.deep) {
        traverse(value)
      }
      // 恢复Watcher
      popTarget()
      // 清理不需要了的依赖
      this.cleanupDeps()
    }
    return value
  }
  // 依赖收集时调用
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        // 把当前 Watcher push 进数组
        dep.addSub(this)
      }
    }
  }
  // 清理不需要的依赖(下面有)
  cleanupDeps () {
    ...
  }
  // 派发更新时调用(下面有)
  update () {
    ...
  }
  // 执行 watcher 的回调
  run () {
    ...
  }
  depend () {
    let i = this.deps.length
    while (i--) {
      this.deps[i].depend()
    }
  }
}
```

补充：

1. 我们自己组件里写的 watch，为什么自动就能拿到新值和老值两个参数？
   就是在 watcher.run() 里面会执行回调，并且把新值和老值传过去

2. 为什么要初始化两个 Dep 实例数组
   因为 Vue 是数据驱动的，每次数据变化都会重新 render，也就是说 vm.render() 方法就又会重新执行，再次触发 getter，所以用两个数组表示，新添加的 Dep 实例数组 newDeps 和上一次添加的实例数组 deps

### 依赖收集过程

在首次渲染挂载的时候，还会有这样一段逻辑

`mountComponent` 源码地址：`src/core/instance/lifecycle.js - 141行`

```js
export function mountComponent (...): Component {
  // 调用生命周期钩子函数
  callHook(vm, 'beforeMount')
  let updateComponent
  updateComponent = () => {
    // 调用 _update 对 render 返回的虚拟 DOM 进行 patch（也就是 Diff )到真实DOM，这里是首次渲染
    vm._update(vm._render(), hydrating)
  }
  // 为当前组件实例设置观察者，监控 updateComponent 函数得到的数据，下面有介绍
  new Watcher(vm, updateComponent, noop, {
    // 当触发更新的时候，会在更新之前调用
    before () {
      // 判断 DOM 是否是挂载状态，就是说首次渲染和卸载的时候不会执行
      if (vm._isMounted && !vm._isDestroyed) {
        // 调用生命周期钩子函数
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  // 没有老的 vnode，说明是首次渲染
  if (vm.$vnode == null) {
    vm._isMounted = true
    // 调用生命周期钩子函数
    callHook(vm, 'mounted')
  }
  return vm
}
```

依赖收集：

- 挂载之前会实例化一个渲染 `watcher` ，进入 `watcher` 构造函数里就会执行 `this.get()` 方法
- 然后就会执行 `pushTarget(this)`，就是把 `Dep.target` 赋值为当前渲染 `watcher` 并压入栈(为了恢复用)
- 然后执行 `this.getter.call(vm, vm)`，也就是上面的 updateComponent() 函数，里面就执行了 `vm._update(vm._render(), hydrating)`
- 接着执行 `vm._render()` 就会生成渲染 `vnode`，这个过程中会访问 vm 上的数据，就触发了数据对象的 getter
- 每一个对象值的 getter 都有一个 `dep`，在触发 getter 的时候就会调用 `dep.depend()` 方法，也就会执行 `Dep.target.addDep(this)`
- 然后这里会做一些判断，以确保同一数据不会被多次添加，接着把符合条件的数据 push 到 subs 里，到这就已经**完成了依赖的收集**，不过到这里还没执行完，如果是对象还会递归对象触发所有子项的 getter，还要恢复 Dep.target 状态

### 移除订阅

移除订阅就是调用 cleanupDeps() 方法。比如在模板中有 v-if 我们收集了符合条件的模板 a 里的依赖。当条件改变时，模板 b 显示出来，模板 a 隐藏。这时就需要移除 a 的依赖

这里主要做的是：

- 先遍历上一次添加的实例数组 deps，移除 dep.subs 数组中的 Watcher 的订阅
- 然后把 newDepIds 和 depIds 交换，newDeps 和 deps 交换
- 再把 newDepIds 和 newDeps 清空

```js
// 清理不需要的依赖
  cleanupDeps () {
    let i = this.deps.length
    while (i--) {
      const dep = this.deps[i]
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this)
      }
    }
    let tmp = this.depIds
    this.depIds = this.newDepIds
    this.newDepIds = tmp
    this.newDepIds.clear()
    tmp = this.deps
    this.deps = this.newDeps
    this.newDeps = tmp
    this.newDeps.length = 0
  }
```

## 派发更新

### notify()

触发 setter 的时候会调用 `dep.notify()` 通知所有订阅者进行派发更新

```js
notify () {
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // 如果不是异步，需要排序以确保正确触发
      subs.sort((a, b) => a.id - b.id)
    }
    // 遍历所有 watcher 实例数组
    for (let i = 0, l = subs.length; i < l; i++) {
      // 触发更新
      subs[i].update()
    }
  }
```

### update()

触发更新时调用

```js
  update () {
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      // 组件数据更新会走这里
      queueWatcher(this)
    }
  }
```

### queueWatcher()

源码地址：`src/core/observer/scheduler.js - 164行`

这是一个队列，也是 Vue 在做派发更新时的一个优化点。就是说在每次数据改变的时候不会都触发 watcher 回调，而是把这些 watcher 都添加到一个队列里，然后在 nextTick 后才执行

这里和下一小节 `flushSchedulerQueue()` 的逻辑有交叉的地方，所以要联合起来理解

主要做的是：

- 先用 has 对象查找 id，保证同一个 watcher 只会 push 一次
- else 如果在执行 watcher 期间又有新的 watcher 插入进来就会到这里，然后从后往前找，找到第一个待插入的 id 比当前队列中的 id 大的位置，插入到队列中，这样队列的长度就发生了变化
- 最后通过 waiting 保证 nextTick 只会调用一次

```js
export function queueWatcher(watcher: Watcher) {
  // 获得 watcher 的 id
  const id = watcher.id
  // 判断当前 id 的 watcher 有没有被 push 过
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      // 最开始会进入这里
      queue.push(watcher)
    } else {
      // 在执行下面 flushSchedulerQueue 的时候，如果有新派发的更新会进入这里，插入新的 watcher，下面有介绍
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // 最开始会进入这里
    if (!waiting) {
      waiting = true
      if (process.env.NODE_ENV !== "production" && !config.async) {
        flushSchedulerQueue()
        return
      }
      // 因为每次派发更新都会引起渲染，所以把所有 watcher 都放到 nextTick 里调用
      nextTick(flushSchedulerQueue)
    }
  }
}
```

### flushSchedulerQueue()

源码地址：`src/core/observer/scheduler.js - 71行`

这里主要做的是：

- 先排序队列，排序条件有三点，看注释
- 然后遍历队列，执行对应 watcher.run()。需要注意的是，遍历的时候每次都会对队列长度进行求值，因为在 run 之后，很可能又会有新的 watcher 添加进来，这时就会再次执行到上面的 queueWatcher

```js
function flushSchedulerQueue() {
  currentFlushTimestamp = getNow()
  flushing = true
  let watcher, id

  // 根据 id 排序，有如下条件
  // 1.组件更新需要按从父到子的顺序，因为创建过程中也是先父后子
  // 2.组件内我们自己写的 watcher 优先于渲染 watcher
  // 3.如果某组件在父组件的 watcher 运行期间销毁了，就跳过这个 watcher
  queue.sort((a, b) => a.id - b.id)

  // 不要缓存队列长度，因为遍历过程中可能队列的长度发生变化
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      // 执行 beforeUpdate 生命周期钩子函数
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    // 执行组件内我们自己写的 watch 的回调函数并渲染组件
    watcher.run()
    // 检查并停止循环更新，比如在 watcher 的过程中又重新给对象赋值了，就会进入无限循环
    if (process.env.NODE_ENV !== "production" && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(`无限循环了`)
        break
      }
    }
  }
  // 重置状态之前，先保留一份队列备份
  const activatedQueue = activatedChildren.slice()
  const updatedQueue = queue.slice()
  resetSchedulerState()
  // 调用组件激活的钩子  activated
  callActivatedHooks(activatedQueue)
  // 调用组件更新的钩子  updated
  callUpdatedHooks(updatedQueue)
}
```

### updated()

终于可以更新了，`updated` 大家都熟悉了，就是生命周期钩子函数

上面调用 `callUpdatedHooks()` 的时候就会进入这里， 执行 `updated` 了

```js
function callUpdatedHooks(queue) {
  let i = queue.length
  while (i--) {
    const watcher = queue[i]
    const vm = watcher.vm
    if (vm._watcher === watcher && vm._isMounted && !vm._isDestroyed) {
      callHook(vm, "updated")
    }
  }
}
```

至此 Vue2 的响应式原理流程的源码基本就分析完毕了，接下来就介绍一下上面流程中的不足之处

## defineProperty 缺陷及处理

使用 Object.defineProperty 实现响应式对象，还是有一些问题的

- 比如给对象中添加新属性时，是无法触发 setter 的
- 比如不能检测到数组元素的变化

而这些问题，Vue2 里也有相应的解决文案

### Vue.set()

给对象添加新的响应式属性时，可以使用一个全局的 API，就是 Vue.set() 方法

源码地址：`src/core/observer/index.js - 201行`

set 方法接收三个参数：

- target：数组或普通对象
- key：表示数组下标或对象的 key 名
- val：表示要替换的新值

这里主要做的是：

- 先判断如果是数组，并且下标合法，就直接使用重写过的 splice 替换
- 如果是对象，并且 key 存在于 target 里，就替换值
- 如果没有 `__ob__`，说明不是一个响应式对象，直接赋值返回
- 最后再把新属性变成响应式，并派发更新

```js
export function set(target: Array<any> | Object, key: any, val: any): any {
  if (
    process.env.NODE_ENV !== "production" &&
    (isUndef(target) || isPrimitive(target))
  ) {
    warn(
      `Cannot set reactive property on undefined, null, or primitive value: ${(target: any)}`
    )
  }
  // 如果是数组 而且 是合法的下标
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.length = Math.max(target.length, key)
    // 直接使用 splice 就替换，注意这里的 splice 不是原生的，所以才可以监测到，具体看下面
    target.splice(key, 1, val)
    return val
  }
  // 到这说明是对象
  // 如果 key 存在于 target 里，就直接赋值，也是可以监测到的
  if (key in target && !(key in Object.prototype)) {
    target[key] = val
    return val
  }
  // 获取 target.__ob__
  const ob = (target: any).__ob__
  if (target._isVue || (ob && ob.vmCount)) {
    process.env.NODE_ENV !== "production" &&
      warn(
        "Avoid adding reactive properties to a Vue instance or its root $data " +
          "at runtime - declare it upfront in the data option."
      )
    return val
  }
  // 在 Observer 里介绍过，如果没有这个属性，就说明不是一个响应式对象
  if (!ob) {
    target[key] = val
    return val
  }
  // 然后把新添加的属性变成响应式
  defineReactive(ob.value, key, val)
  // 手动派发更新
  ob.dep.notify()
  return val
}
```

### 重写数组方法

源码地址：`src/core/observer/array.js`

这里做的主要是：

- 保存会改变数组的方法列表
- 当执行列表里有的方法的时候，比如 push，先把原本的 push 保存起来，再做响应式处理，再执行这个方法

```js
// 获取数组的原型
const arrayProto = Array.prototype
// 创建继承了数组原型的对象
export const arrayMethods = Object.create(arrayProto)
// 会改变原数组的方法列表
const methodsToPatch = [
  "push",
  "pop",
  "shift",
  "unshift",
  "splice",
  "sort",
  "reverse",
]
// 重写数组事件
methodsToPatch.forEach(function (method) {
  // 保存原本的事件
  const original = arrayProto[method]
  // 创建响应式对象
  def(arrayMethods, method, function mutator(...args) {
    const result = original.apply(this, args)
    const ob = this.__ob__
    let inserted
    switch (method) {
      case "push":
      case "unshift":
        inserted = args
        break
      case "splice":
        inserted = args.slice(2)
        break
    }
    if (inserted) ob.observeArray(inserted)
    // 派发更新
    ob.dep.notify()
    // 做完我们需要的处理后，再执行原本的事件
    return result
  })
})
```
