---
title: Vue2完整生命周期流程
date: 2021-01-02 21:34:30
author: 沐华
img: https://cdn.jsdelivr.net/gh/wmuhua/cdn@main/blog/21.jpg
top: false
cover: true
coverImg: /images/1.jpg
password:
toc: false
mathjax: false
summary: 本文主要梳理一下Vue2生命周期完整流程
categories: Vue
tags:
  - Vue
  - 生命周期
  - 源码
---

本文源码版本 `2.6.14`，Vue3 源码剖析看我另一篇文章 [Vue3.2 响应式原理源码剖析，及与 Vue2 .x 响应式的区别](https://juejin.cn/post/7021046293627666463)

请说一下 Vue 的生命周期？这种烂大街的问题为什么还在问？

- 考察你的熟练度
- 考察你的深度
- 考察你的知识面

你说是吗，关于 Vue 生命周期有些能说出下面的钩子函数名，有些甚至这些钩子函数名都说不上来，那是真的需要补充一下了，因为这些钩子函数也只是 Vue 完整生命周期中的冰山一角

源码地址：`src/shared/constants.js - 9行`

```js
export const LIFECYCLE_HOOKS = [
  "beforeCreate",
  "created",
  "beforeMount",
  "mounted",
  "beforeUpdate",
  "updated",
  "beforeDestroy",
  "destroyed",
  "activated",
  "deactivated",
  "errorCaptured",
  "serverPrefetch",
]
```

Vue2 完整的生命周期大致可分成四个阶段

- 初始化阶段：为 Vue 实例初始化一些事件、属性和响应式数据等
- 模板编译阶段：把我们写的 `<template></template>` 模板编译成渲染函数 `render`
- 挂载阶段：把模板渲染到真实的 DOM 节点上，以及数据变更时执行更新操作
- 销毁阶段：把组件实例从父组件中删除，并取消依赖监听和事件监听

这四个阶段分别是怎么划分的，来看一张图（图片来源于 Vue 中文社区）

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b1493c640d7e4cf2bd7785cea7c86789~tplv-k3u1fbpfcp-watermark.image?)

下面再从源码为大家一一展开每一个阶段做的事，注意看每个生命周期钩子函数调用的时间，以及前后都分别做了什么

## 初始化阶段

### new Vue()

这个阶段做的第一件事，就是用 new 创建一个 Vue 实例对象

```js
new Vue({
  el: "#app",
  store,
  router,
  render: (h) => h(App),
})
```

能用 new 那肯定是有一个构造函数的，我们来看一下

源码地址：`src/core/instance/index.js - 8行`

```js
function Vue(options) {
  if (process.env.NODE_ENV !== "production" && !(this instanceof Vue)) {
    warn("Vue is a constructor and should be called with the `new` keyword")
  }
  this._init(options)
}
initMixin(Vue)
```

可以看出 new Vue() 的关键就是 `_init()` 了，来看一下它是哪来的，干了些什么

### \_init()

源码地址：`src/core/instance/init.js - 15行`

我这里去掉了一些环境判断的，主要流程就是

- 合并配置，主要是把一些内置组件`<component/>`、`<keep-alive>`、`<transition>`、`directive`、`filter`、本文最开始的钩子函数名称列表等合并到 Vue.options 上面
- 调用一些初始化函数，这里具体初始化了哪些东西我写在注释里了
- 触发生命周期钩子，`beforeCreate` 和 `created`
- 最后调用 `$mount` 挂载 进入下一阶段

```js
export function initMixin(Vue: Class<Component>) {
  // 在原型上添加 _init 方法
  Vue.prototype._init = function (options?: Object) {
    // 保存当前实例
    const vm: Component = this
    // 合并配置
    if (options && options._isComponent) {
      // 把子组件依赖父组件的 props、listeners 挂载到 options 上，并指定组件的$options
      initInternalComponent(vm, options)
    } else {
      // 把我们传进来的 options 和当前构造函数和父级的 options 进行合并，并挂载到原型上
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    vm._self = vm
    initLifecycle(vm) // 初始化实例的属性、数据：$parent, $children, $refs, $root, _watcher...等
    initEvents(vm) // 初始化事件：$on, $off, $emit, $once
    initRender(vm) // 初始化渲染： render, mixin
    callHook(vm, "beforeCreate") // 调用生命周期钩子函数
    initInjections(vm) // 初始化 inject
    initState(vm) // 初始化组件数据：props, data, methods, watch, computed
    initProvide(vm) // 初始化 provide
    callHook(vm, "created") // 调用生命周期钩子函数

    if (vm.$options.el) {
      // 如果传了 el 就会调用 $mount 进入模板编译和挂载阶段
      // 如果没有传就需要手动执行 $mount 才会进入下一阶段
      vm.$mount(vm.$options.el)
    }
  }
}
```

这里需要记着上面流程说明，合并了哪些配置，初始化了什么东西，分别在什么时候调用了两个生命周期钩子函数，最后调用 `$mount` 挂载，进入下一阶段

## 模板编译阶段

先看图了解一下这个阶段做的事

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aaa4b9bb711b450bbf04d3d3c22c020b~tplv-k3u1fbpfcp-watermark.image?)

### $mount()

源码地址：`dist/vue.js - 11927行`

```js
Vue.prototype.$mount = function (el, hydrating) {
  el = el && query(el)
  var options = this.$options
  // 如果没有 render
  if (!options.render) {
    var template = options.template
    // 再判断，如果有 template
    if (template) {
      if (typeof template === "string") {
        if (template.charAt(0) === "#") {
          template = idToTemplate(template)
        }
      } else if (template.nodeType) {
        template = template.innerHTML
      } else {
        return this
      }
      // 再判断，如果有 el
    } else if (el) {
      template = getOuterHTML(el)
    }
  }
  return mount.call(this, el, hydrating)
}
```

说白了 `$mount` 主要就是判断要不要编译，使用哪一个模板编译，需要注意的就是判断顺序了，我们来看一下这段代码

```js
<div id='app'>
    <p>{{ name }}</p>
</div>
<script>
    new Vue({
        el:'#app',
        data:{ name:'沐华' },
        template:'<div>掘金</div>',
        render(h){
            return h('div', {}, '好好学习，天天向上')
        }
    })
</script>
```

可能有点别扭，可以想象一下执行上面这样的代码，会渲染出什么

结合源码马上就知道会渲染出来的只有 `<div>好好学习，天天向上</div>`

因为源码中是优先判断 `render` 是否存在，如果存在，就直接使用 render 函数了

如果没有，再判断 template 和 el，如果有 `template`，就不会管 `el` 了

所以优先级顺序是：`render > template > el`

因为不管是 `el` 挂载的，还是 `template` 最后都会被编译成 `render` 函数，而如果已经有了 `render` 函数了，就跳过前面的编译了

关于模板编译是怎么编译的？render 是怎么来的？更多详细内容有点多，具体可以看我另一篇文章有源码的完整流程详细介绍 [render 函数是怎么来的？深入浅出 Vue 中的模板编译](https://juejin.cn/post/7011294489335562247)

得到 render 函数之后，就会进入挂载阶段了

## 挂载阶段

先看图了解一下这个阶段做的事

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/13d0a584fb8341c2a4b32788be3ace0e~tplv-k3u1fbpfcp-watermark.image?)

如图也可以看得出来，这里阶段主要做的事有两件

1. 根据 render 返回的虚拟 DOM 创建真实的 DOM 节点，插入到视图中，完成渲染
2. 对模板中数据或状态做响应式处理

### 挂载 mountComponent()

我们来看挂载的源码，这里我删除了一点关于环境判断的，方便看主要流程

这里主要做的事就是

- 调用钩子函数 `beforeMount`
- 调用 `_update()` 方法对新老虚拟 DOM 进行 `patch` 以及 `new Watcher` 对模板数据做响应式处理
- 再调用钩子函数 `mounted`

源码地址：`src/core/instance/lifecycle.js - 141行`

```js
export function mountComponent(
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el
  // 判断有没有渲染函数 render
  if (!vm.$options.render) {
    // 如果没有，默认就创建一个注释节点
    vm.$options.render = createEmptyVNode
  }
  // 调用生命周期钩子函数
  callHook(vm, "beforeMount")
  let updateComponent
  updateComponent = () => {
    // 调用 _update 对 render 返回的虚拟 DOM 进行 patch（也就是 Diff )到真实DOM，这里是首次渲染
    vm._update(vm._render(), hydrating)
  }
  // 为当前组件实例设置观察者，监控 updateComponent 函数得到的数据，下面有介绍
  new Watcher(
    vm,
    updateComponent,
    noop,
    {
      // 当触发更新的时候，会在更新之前调用
      before() {
        // 判断 DOM 是否是挂载状态，就是说首次渲染和卸载的时候不会执行
        if (vm._isMounted && !vm._isDestroyed) {
          // 调用生命周期钩子函数
          callHook(vm, "beforeUpdate")
        }
      },
    },
    true /* isRenderWatcher */
  )
  hydrating = false
  // 没有老的 vnode，说明是首次渲染
  if (vm.$vnode == null) {
    vm._isMounted = true
    // 调用生命周期钩子函数
    callHook(vm, "mounted")
  }
  return vm
}
```

更新

### Watcher

关于响应式原理，可以看我另一篇文章介绍的很详细了就不搬过来了，[深入浅出 Vue 响应式原理源码剖析](https://juejin.cn/post/7017327623307001864)

### \_update()

然后再了解一下去 patch 的入口，有关 Diff 算法源码的完整流程剖析，可以看我另一篇文章介绍的很详细了就不搬过来了， [深入浅出虚拟 DOM 和 Diff 算法，及 Vue2 与 Vue3 中的区别](https://juejin.cn/post/7010594233253888013)

源码地址：`src/core/instance/lifecycle.js - 59行`

```js
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this
  const prevEl = vm.$el // 当前组件根节点
  const prevVnode = vm._vnode // 老的 vnode
  vm._vnode = vnode // 更新老的 vnode
  // 如果是首次渲染
  if (!prevVnode) {
    // 对 vnode 进行 patch 创建真实的 DOM，挂载到 vm.$el 上
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
  } else {
    // 修改的时候，进行新老 vnode 对比，并回修改后的真实 DOM
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  // 删除老根节点的引用
  if (prevEl) prevEl.__vue__ = null
  // 更新当前根节点的引用
  if (vm.$el) vm.$el.__vue__ = vm
  // 更新父级的引用
  if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
    vm.$parent.$el = vm.$el
  }
}
```

## 销毁阶段

先看图了解一下这个阶段做的事

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/73aa1bfc48b5475cabb906557c5d9b10~tplv-k3u1fbpfcp-watermark.image?)

就是说在调用 `vm.$destroy()` 方法的时候，Vue 就进入销毁阶段了，来看一下组件销毁过程都干了些啥，是怎么销毁的？

### $destroy()

这个阶段比较简单，源码不多，主要就是：

- 调用生命周期钩子函数 `beforeDestory`
- 从父组件中删除当前组件
- 移除当前组件内的所有观察者(依赖追踪)，删除数据对象的引用，删除虚拟 DOM
- 调用生命周期钩子函数 `destoryed`
- 关闭所有事件监听，删除当前根组件的引用，删除父级的引用

每一行我都写在注释里了，注意看

源码地址：`src/core/instance/lifecycle.js - 97行`

```js
Vue.prototype.$destroy = function () {
  const vm: Component = this
  // 如果实例正在被销毁的过程中，直接跳过
  if (vm._isBeingDestroyed) {
    return
  }
  // 调用生命周期钩子函数
  callHook(vm, "beforeDestroy")
  // 更新销毁过程状态
  vm._isBeingDestroyed = true
  // 获取父级
  const parent = vm.$parent
  // 如果父级存在，并且父级没有在被销毁，并且不是抽象组件而是真实组件(<keep-alive>就是抽象组件，它的abstract就为true)
  if (parent && !parent._isBeingDestroyed && !vm.$options.abstract) {
    // 从父级中删除当前组件
    remove(parent.$children, vm)
  }
  // 移除实例的所有观察者
  if (vm._watcher) {
    // 删除实例自身依赖
    vm._watcher.teardown()
  }
  let i = vm._watchers.length
  while (i--) {
    // 删除实例内数据对其他数据的依赖
    vm._watchers[i].teardown()
  }
  // 删除数据对象的引用
  if (vm._data.__ob__) {
    vm._data.__ob__.vmCount--
  }
  // 更新组件销毁状态
  vm._isDestroyed = true
  // 删除实例的虚拟 DOM
  vm.__patch__(vm._vnode, null)
  // 调用生命周期钩子函数
  callHook(vm, "destroyed")
  // 关闭所有事件监听
  vm.$off()
  // 删除当前根组件的引用
  if (vm.$el) {
    vm.$el.__vue__ = null
  }
  // 删除父级的引用
  if (vm.$vnode) {
    vm.$vnode.parent = null
  }
}
```

### 什么时候会调用 $destroy()

我在 patch 源码里看到有三次调用，源码地址：`src/core/vdom/patch.js`

- 新的 vnode 不存在，老的 vnode 存在的时候，就触发卸载老 vnode 对应组件的 destroy。702 行
- 如果新的 vnode 根节点被修改的时候，调用老 vnode 对应组件的 destroy。767 行
- 新老 vnode 对比结束后，调用老 vnode 对应组件的 destroy。795 行

至此，一轮完整的 Vue 生命周期就走完了
