---
title: Vue3的几个知识点
date: 2020-06-12 22:39:06
author: 沐华
img: https://cdn.jsdelivr.net/gh/wmuhua/cdn@main/blog/26.jpg
top: false
cover: true
coverImg: /images/1.jpg
password:
toc: false
mathjax: false
summary: 本文主要是总结了几个 Vue3 的知识点
categories: Vue
tags:
  - Vue3
---

前一段时间一直在研究 Vue 源码，2 和 3 的源码变化还是挺大的，但是 Vue3 在开发上需要改变以往习惯的地方真没什么，毕竟基础概念是一模一样的，可偏偏 3 的项目写起来就是能让人感觉爽得多，不动手是真体会不到

所以咯，一定要动手！敲代码！不能学而不用，这是不可取的

这里我总结了几个 Vue3 的知识点，如果你是 Vue2 的迁移者、学习 Vue3 或者准备面试的话，相信看完本文一定会有所收获

## Vue3 有哪些变化

Vue3 是怎么得更快的？

- 新增了三个组件：`Fragment` 支持多个根节点、`Suspense` 可以在组件渲染之前的等待时间显示指定内容、`Teleport` 可以让子组件能够在视觉上跳出父组件(如父组件 overflow:hidden)
- 新增指令 `v-memo`，可以缓存 html 模板，比如 v-for 列表不会变化的就缓存，简单说就是用内存换时间
- 支持 `Tree-Shaking`，会在打包时去除一些无用代码，没有用到的模块，使得代码打包体积更小
- 新增 `Composition API` 可以更好的逻辑复用和代码组织，同一功能的代码不至于像以前一样太分散，虽然 Vue2 中可以用 minxin 来实现复用代码，但也存在问题，比如方法或属性名会冲突，代码来源也不清楚等
- 用 `Proxy` 代替 Object.defineProperty 重构了响应式系统，可以监听到数组下标变化，及对象新增属性，因为监听的不是对象属性，而是对象本身，还可拦截 apply、has 等 13 种方法
- 重构了虚拟 DOM，在编译时会将事件缓存、将 slot 编译为 lazy 函数、保存静态节点直接复用(静态提升)、以及添加静态标记、Diff 算法使用 最长递增子序列 优化了对比流程，使得虚拟 DOM 生成速度提升 `200%`
- 支持在 `<style></style>` 里使用 `v-bind`，给 CSS 绑定 JS 变量(`color: v-bind(str)`)
- 删除了两个生命周期 beforeCreate、created，直接用 `setup` 代替了
- 新增了**开发环境**的两个钩子函数，在组件更新时 `onRenderTracked` 会跟踪组件里所有变量和方法的变化、每次触发渲染时 `onRenderTriggered` 会返回发生变化的新旧值，可以让我们进行有针对性调试
- 毕竟 Vue3 是用 `TS` 写的，所以对 `TS` 的支持度更好
- Vue3 不兼容 `IE11`

## 说一下 Composition API

和 Options API 的区别？

`Composition API` 也叫**组合式 API**，它主要就是为了解决 Vue2 中 Options API 的问题。

一是在 Vue2 中只能固定用 `data`、`computed`、`methods` 等选项组织代码，在组件越来越复杂的时候，一个功能相关的属性和方法就会在文件上中下到处都有，很分散，变越来越难维护

二是 Vue2 中虽然可以用 `minxin` 来做逻辑的提取复用，但是 minxin 里的属性和方法名会和组件内部的命名冲突，还有当引入多个 minxin 的时候，我们使用的属性或方法是来于哪个 minxin 也不清楚

而 `Composition API` 刚才就解决了这两个问题，可以让我们自由的组织代码，同一功能相关的全部放在一起，代码有更好的可读性更便于维护，单独提取出来也不会造成命名冲突，所以也有更好的可扩展性

## 说一下 setup

```js
// 方法
setup(props, context){ return { name:'沐华' } }

// 语法糖
<script setup> ... </script>
```

`setup()` 方法是在 `beforeCreate()` 生命周期函数之前执行的函数；

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3f6372e90f1f41e3b95b9a69da505470~tplv-k3u1fbpfcp-watermark.image)

它接收两个参数 `props` 和 `context`。它里面不能使用 `this`，而是通过 context 对象来代替当前执行上下文绑定的对象，context 对象有四个属性：`attrs`、`slots`、`emit`、`expose`

里面通过 `ref` 和 `rective` 代替以前的 data 语法，`return` 出去的内容，可以在模板直接使用，包括变量和方法

而使用 `setup` 语法糖时，不用写 `export default {}`，子组件只需要 `import` 就直接使用，不需要像以前一样在 components 里注册，属性和方法也不用 return。

并且里面不需要用 `async` 就可以直接使用 `await`，因为这样默认会把组件的 `setup` 变为 `async setup`

用语法糖时，props、attrs、slots、emit、expose 的获取方式也不一样了

3.0~3.2 版本变成了通过 import 引入的 API：`defineProps`、`defineEmit`、`useContext`(在 3.2 版本已废弃)，useContext 的属性 `{ emit, attrs, slots, expose }`

3.2+版本不需要引入，而直接调用：`defineProps`、`defineEmits`、`defineExpose`、`useSlots`、`useAttrs`

## watch 和 watchEffect 的区别

`watch` 作用是对传入的某个或多个值的变化进行监听；触发时会返回新值和老值；也就是说第一次不会执行，只有变化时才会重新执行

```js
const name = ref('沐华')
watch(name, (newValue, oldValue)=>{ ... }, {immediate:true, deep:true})

// 响应式对象
const boy = reactive({ age:18 })
watch(()=>boy.age, (newValue, oldValue)=>{ ... })

// 监听多个
watch( [name, ()=>boy.age], (newValue, oldValue)=>{ ... } )
```

`watchEffect` 是传入一个立即执行函数，所以默认第一次也会执行一次；不需要传入监听内容，会自动收集函数内的数据源作为依赖，在依赖变化的时候又会重新执行该函数，如果没有依赖就不会执行；而且不会返回变化前后的新值和老值

```js
watchEffect(onInvalidate =>{ ... })
```

共同点是 `watch` 和 `watchEffect` 会共享以下四种行为：

- `停止监听`：组件卸载时都会自动停止监听
- `清除副作用`：onInvalidate 会作为回调的第三个参数传入
- `副作用刷新时机`：响应式系统会缓存副作用函数，并异步刷新，避免同一个 tick 中多个状态改变导致的重复调用
- `监听器调试`：开发模式下可以用 onTrack 和 onTrigger 进行调试

## Vue3 响应式原理和 Vue2 的区别

众所周知 `Vue2` 数据响应式是通过 `Object.defineProperty()` 劫持各个属性 getter 和 setter，在数据变化时发布消息给订阅者，触发相应的监听回调，而这之间存在几个问题

- 初始化时需要遍历对象所有 key，如果对象层次较深，性能不好
- 通知更新过程需要维护大量 dep 实例和 watcher 实例，额外占用内存较多
- Object.defineProperty 无法监听到数组元素的变化，只能通过劫持重写数方法
- 动态新增，删除对象属性无法拦截，只能用特定 set/delete API 代替
- 不支持 Map、Set 等数据结构

而在 `Vue3` 中为了解决这些问题，使用原生的 `proxy` 代替，支持监听对象和数组的变化，并且多达 13 种拦截方法，动态属性增删都可以拦截，新增数据结构全部支持，对象嵌套属性只代理第一层，运行时递归，用到才代理，也不需要维护特别多的依赖关系，性能取得很大进步

更多细节和源码剖析可以看我另一篇文章 [Vue3.2 响应式原理源码剖析，及与 Vue2 响应式的区别](https://juejin.cn/post/7021046293627666463)

## defineProperty 和 Proxy 的区别

为什么要用 Proxy 代替 defineProperty ？好在哪里？

- Object.defineProperty 是 Es5 的方法，Proxy 是 Es6 的方法
- defineProperty 不能监听到数组下标变化和对象新增属性，Proxy 可以
- defineProperty 是劫持对象属性，Proxy 是代理整个对象
- defineProperty 局限性大，只能针对单属性监听，所以在一开始就要全部递归监听。Proxy 对象嵌套属性运行时递归，用到才代理，也不需要维护特别多的依赖关系，性能提升很大，且首次渲染更快
- defineProperty 会污染原对象，修改时是修改原对象，Proxy 是对原对象进行代理并会返回一个新的代理对象，修改的是代理对象
- defineProperty 不兼容 IE8，Proxy 不兼容 IE11

## Vue3 的生命周期

基本上就是在 Vue2 生命周期钩子函数名基础上加了 `on`；beforeDestory 和 destoryed 更名为 onBeforeUnmount 和 onUnmounted；然后删了两个钩子函数 beforeCreate 和 created；新增了两个开发环境用于调试的钩子

![未命名文件 (1).png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/60f518fa89ba44bb8b59911ecd65a8a4~tplv-k3u1fbpfcp-watermark.image?)

## Vue3 的 8 种组件通信

可以看我另一文章，就不搬过来了 [Vue3 的 8 种和 Vue2 的 12 种组件通信，值得收藏](https://juejin.cn/post/6999687348120190983)

## Vue3 Diff 算法和 Vue2 的区别

我们知道在数据变更触发页面重新渲染，会生成虚拟 DOM 并进行 patch 过程，这一过程在 Vue3 中的优化有如下

编译阶段的优化：

- 事件缓存：将事件缓存(如: @click)，可以理解为变成静态的了
- 静态提升：第一次创建静态节点时保存，后续直接复用
- 添加静态标记：给节点添加静态标记，以优化 Diff 过程

由于编译阶段的优化，除了能更快的生成虚拟 DOM 以外，还使得 Diff 时可以跳过"永远不会变化的节点"，Diff 优化如下

- Vue2 是全量 Diff，Vue3 是静态标记 + 非全量 Diff
- 使用最长递增子序列优化了对比流程

根据尤大公布的数据就是 Vue3 `update` 性能提升了 `1.3~2 倍`

源码剖析可以看我另一篇文章 [深入浅出虚拟 DOM 和 Diff 算法，及 Vue2 与 Vue3 中的区别](https://juejin.cn/post/7010594233253888013)
