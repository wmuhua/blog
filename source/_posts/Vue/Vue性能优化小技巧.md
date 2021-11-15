---
title: Vue性能优化小技巧
date: 2021-03-25 05:41:01
author: 沐华
img: https://cdn.jsdelivr.net/gh/wmuhua/cdn@main/blog/27.jpg
top: false
cover: true
coverImg: /images/1.jpg
password:
toc: false
mathjax: false
summary: 性能优化，是我们都会遇到的问题
categories: Vue
tags:
  - Vue
  - 性能优化
---

性能优化，是每一个开发者都会遇到的问题，特别是现在越来越重视体验，以及竞争越来越激烈的环境下，对于我们开发者来说，只完成迭代，把功能做好是远远不够的，最重要的是把产品做好，让更多人愿意使用，让用户用得更爽，这不也是我们开发者价值与能力的体现吗。重视性能问题，优化产品的体验，比起改几个无关痛痒的 bug 要有价值得多

本文记录了我在 Vue 项目日常开发中的一些小技巧，废话不多说，我们开始吧

## 1. 长列表性能优化

### 1. 不做响应式

比如会员列表、商品列表之类的，只是纯粹的数据展示，不会有任何动态改变的场景下，就不需要对数据做响应化处理，可以大大提升渲染速度

比如使用 `Object.freeze()` 冻结一个对象，[MDN 的描述是](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze) 该方法冻结的对象不能被修改；即不能向这个对象添加新属性，不能删除已有属性，不能修改该对象已有属性的可枚举性、可配置性、可写性，以及不能修改已有属性的值，以及该对象的原型也不能被修改

```js
export default {
  data: () => ({
    userList: [],
  }),
  async created() {
    const users = await axios.get("/api/users")
    this.userList = Object.freeze(users)
  },
}
```

Vue2 的响应式源码地址：`src/core/observer/index.js - 144行` 是这样的

```js
export function defineReactive (...){
    const property = Object.getOwnPropertyDescriptor(obj, key)
    if (property && property.configurable === false) {
        return
    }
    ...
}
```

可以看到一开始就判断 `configurable` 为 `false` 的直接返回不做响应式处理

`configurable` 为 `false` 表示这个属性是不能被修改的，而冻结的对象的 `configurable` 就是为 `false`

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3b6499584efd49f4a1870acf0154f086~tplv-k3u1fbpfcp-watermark.image?)

Vue3 里则是添加了响应式`flag`，用于标记目标对象类型，Vue3 的响应式详情可以看我另一篇文章

### 2. 虚拟滚动

如果是大数据很长的列表，全部渲染的话一次性创建太多 DOM 就会非常卡，这时就可以用虚拟滚动，只渲染少部分(含可视区域)区域的内容，然后滚动的时候，不断替换可视区域的内容，模拟出滚动的效果

```html
<recycle-scroller class="items" :items="items" :item-size="24">
  <template v-slot="{ item }">
    <FetchItemView :item="item" @vote="voteItem(item)" />
  </template>
</recycle-scroller>
```

参考 [vue-virtual-scroller](https://github.com/Akryum/vue-virtual-scroller)、[vue-virtual-scroll-list](https://github.com/tangbc/vue-virtual-scroll-list)

原理是监听滚动事件，动态更新需要显示的 DOM，并计算出在视图中的位移，这也意味着在滚动过程需要实时计算，有一定成本，所以如果数据量不是很大的情况下，用普通的滚动就行

## 2. v-for 遍历避免同时使用 v-if

为什么要避免同时使用 `v-for` 和 `v-if`

在 Vue2 中 `v-for` 优先级更高，所以编译过程中会把列表元素全部遍历生成虚拟 DOM，再来通过 v-if 判断符合条件的才渲染，就会造成性能的浪费，因为我们希望的是不符合条件的虚拟 DOM 都不要生成

在 Vue3 中 `v-if` 的优先级更高，就意味着当判断条件是 v-for 遍历的列表中的属性的话，v-if 是拿不到的

所以在一些需要同时用到的场景，就可以通过计算属性来过滤一下列表，如下

```js
<template>
    <ul>
      <li v-for="item in activeList" :key="item.id">
        {{ item.title }}
      </li>
    </ul>
</template>
<script>
// Vue2.x
export default {
    computed: {
      activeList() {
        return this.list.filter( item => {
          return item.isActive
        })
      }
    }
}

// Vue3
import { computed } from "vue";
const activeList = computed(() => {
  return list.filter( item => {
    return item.isActive
  })
})
</script>
```

## 3. 列表使用唯一 key

比如有一个列表，我们需要在中间插入一个元素，在不使用 key 或者使用 index 作为 key 会发生什么变化呢？先看个图

![diff3.jpg](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0dee9fb8ad4243e094c68448716fc13e~tplv-k3u1fbpfcp-watermark.image?)

如图的 `li1` 和 `li2` 不会重新渲染，这个没有争议的。而 `li3、li4、li5` 都会重新渲染

因为在不使用 `key` 或者列表的 `index` 作为 `key` 的时候，每个元素对应的位置关系都是 index，上图中的结果直接导致我们插入的元素到后面的全部元素，对应的位置关系都发生了变更，所以在 patch 过程中会将它们全都执行更新操作，再重新渲染。这可不是我们想要的，我们希望的是渲染添加的那一个元素，其他四个元素不做任何变更，也就不要重新渲染

而在使用唯一 `key` 的情况下，每个元素对应的位置关系就是 `key`，来看一下使用唯一 `key` 值的情况下

![diff4.jpg](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1882d22f62b4433aa0a91dcbe0ad09d5~tplv-k3u1fbpfcp-watermark.image?)

这样如图中的 `li3` 和 `li4` 就不会重新渲染，因为元素内容没发生改变，对应的位置关系也没有发生改变。

这也是为什么 v-for 必须要写 key，而且不建议开发中使用数组的 index 作为 key 的原因

## 4. 使用 v-show 复用 DOM

`v-show`：是渲染组件，然后改变组件的 display 为 block 或 none  
`v-if`：是渲染或不渲染组件

所以对于可以频繁改变条件的场景，就使用 v-show 节省性能，特别是 DOM 结构越复杂收益越大

不过它也有劣势，就是 v-show 在一开始的时候，所有分支内部的组件都会渲染，对应的生命周期钩子函数都会执行，而 v-if 只会加载判断条件命中的组件，所以需要根据不同场景使用合适的指令

比如下面的用 `v-show` 复用 DOM，比 `v-if/v-else` 效果好

```html
<template>
  <div>
    <div v-show="status" class="on">
      <my-components />
    </div>
    <section v-show="!status" class="off">
      <my-components >
    </section>
  </div>
</template>
```

原理就是使用 v-if 当条件变化的时候，触发 diff 更新，发现新旧 vnode 不一致，就会移除整个旧的 `vnode`，再重新创建新的 `vnode`，然后创建新的 `my-components` 组件，又会经历组件自身初始化，`render`，`patch` 等过程，而 `v-show` 在条件变化的时候，新旧 `vnode` 是一致的，就不会执行移除创建等一系列流程

## 5. 无状态的组件用函数式组件

对于一些纯展示，没有响应式数据，没有状态管理，也不用生命周期钩子函数的组件，我们就可以设置成函数式组件，提高渲染性能，因为会把它当成一个函数来处理，所以开销很低

原理是在 `patch` 过程中对于函数式组件的 `render` 生成的虚拟 DOM，不会有递归子组件初始化的过程，所以渲染开销会低很多

它可以接受 `props`，但是由于不会创建实例，所以内部不能使用 `this.xx` 获取组件属性，写法如下

```js
<template functional>
  <div>
    <div class="content">{{ value }}</div>
  </div>
</template>
<script>
export default {
  props: ['value']
}
</script>

// 或者
Vue.component('my-component', {
  functional: true, // 表示该组件为函数式组件
  props: { ... }, // 可选
  // 第二个参数为上下文，没有 this
  render: function (createElement, context) {
    // ...
  }
})
```

## 6. 子组件分割

先看个例子

```js
<template>
  <div :style="{ opacity: number / 100 }">
    <div>{{ someThing() }}</div>
  </div>
</template>
<script>
export default {
  props:['number'],
  methods: {
    someThing () { /* 耗时任务 */ }
  }
}
</script>
```

上面这样的代码中，每次父组件传过来的 `number` 发生变化时，每次都会重新渲染，并且重新执行 `someThing` 这个耗时任务

所以优化的话一个是用计算属性，因为计算属性自身有缓存计算结果的特性

第二个是拆分成子组件，因为 Vue 的更新是组件粒度的，虽然第次数据变化都会导致父组件的重新渲染，但是子组件却不会重新渲染，因为它的内部没有任何变化，耗时任务自然也就不会重新执行，因此性能更好，优化代码如下

```js
<template>
  <div>
    <my-child />
  </div>
</template>
<script>
export default {
  components: {
    MyChild: {
      methods: {
        someThing () { /* 耗时任务 */ }
      },
      render (h) {
        return h('div', this.someThing())
      }
    }
  }
}
</script>
```

## 7. 变量本地化

简单说就是把会多次引用的变量保存起来，因为每次访问 `this.xx` 的时候，由于是响应式对象，所以每次都会触发 `getter`，然后执行依赖收集的相关代码，如果使用变量次数越多，性能自然就越差

从需求上说在一个函数里一个变量执行一次依赖收集就够了，可是很多人习惯性的在项目中大量写 `this.xx`，而忽略了 `this.xx` 背后做的事，就会导致性能问题了

比如下面例子

```js
<template>
  <div :style="{ opacity: number / 100 }"> {{ result }}</div>
</template>
<script>
import { someThing } from '@/utils'
export default {
  props: ['number'],
  computed: {
    base () { return 100 },
    result () {
      let base = this.base, number = this.number // 保存起来
      for (let i = 0; i < 1000; i++) {
        number += someThing(base) // 避免频繁引用 this.xx
      }
      return number
    }
  }
}
</script>
```

## 8. 第三方插件按需引入

比如 `Element-UI` 这样的第三方组件库可以按需引入避免体积太大，特别是项目不大的情况下，更没有必要完整引入组件库

```js
// main.js
import Element3 from "plugins/element3"
Vue.use(Element3)

// element3.js
// 完整引入
import element3 from "element3"
import "element3/lib/theme-chalk/index.css"

// 按需引入
// import "element3/lib/theme-chalk/button.css";
// ...
// import {
// ElButton,
// ElRow,
// ElCol,
// ElMain,
// .....
// } from "element3";

export default function (app) {
  // 完整引入
  app.use(element3)

  // 按需引入
  // app.use(ElButton);
}
```

## 9. 路由懒加载

我们知道 Vue 是单页应用，所以如果没有用懒加载，就会导致进入首页时需要加载的内容过多，时间过长，就会出现长时间的白屏，很不利于用户体验，SEO 也不友好

所以可以去用懒加载将页面进行划分，需要的时候才加载对应的页面，以分担首页的加载压力，减少首页加载时间

没有用路由懒加载：

```js
import Home from "@/components/Home"
const router = new VueRouter({
  routes: [{ path: "/home", component: Home }],
})
```

用了路由懒加载：

```js
const router = new VueRouter({
  routes: [
    { path: "/home", component: () => import("@/components/Home") },
    { path: "/login", component: require("@/components/Home").default },
  ],
})
```

在进入这个路由的时候才会走对应的 `component`，然后运行 `import` 编译加载组件，可以理解为 `Promise` 的 `resolve` 机制

- `import`：Es6 语法规范、编译时调用、是解构过程、不支持变量函数等
- `require`：AMD 规范、运行时调用、是赋值过程，支持变量计算函数等

更多有关前端模块化的内容可以看我另一篇文章 [前端模块化规范详细总结](https://juejin.cn/post/6996595779037036580)

## 10. keep-alive 缓存页面

比如在表单输入页面进入下一步后，再返回上一步到表单页时要保留表单输入的内容、比如在`列表页>详情页>列表页`，这样来回跳转的场景等

我们都可以通过内置组件 `<keep-alive></keep-alive>` 来把组件缓存起来，在组件切换的时候不进行卸载，这样当再次返回的时候，就能从缓存中快速渲染，而不是重新渲染，以节省性能

只需要包裹想要缓存的组件即可

```html
<template>
  <div id="app">
    <keep-alive>
      <router-view />
    </keep-alive>
  </div>
</template>
```

- 也可以用 `include/exclude` 来 缓存/不缓存 指定组件
- 可通过两个生命周期 `activated/deactivated` 来获取当前组件状态

## 11. 事件的销毁

Vue 组件销毁时，会自动解绑它的全部指令及事件监听器，但是仅限于组件本身的事件

而对于`定时器`、
`addEventListener` 注册的监听器等，就需要在组件销毁的生命周期钩子中手动销毁或解绑，以避免内存泄露

```js
<script>
export default {
    created() {
      this.timer = setInterval(this.refresh, 2000)
      addEventListener('touchmove', this.touchmove, false)
    },
    beforeDestroy() {
      clearInterval(this.timer)
      this.timer = null
      removeEventListener('touchmove', this.touchmove, false)
    }
}
</script>
```

## 12. 图片懒加载

图片懒加载就是对于有很多图片的页面，为了提高页面加载速度，只加载可视区域内的图片，可视区域外的等到滚动到可视区域后再去加载

这个功能一些 UI 框架都有自带的，如果没有呢？

推荐一个第三方插件 `vue-lazyload`

```js
npm i vue-lazyload -S

// main.js
import VueLazyload from 'vue-lazyload'
Vue.use(VueLazyload)

// 接着就可以在页面中使用 v-lazy 懒加载图片了
<img v-lazy="/static/images/1.png">
```

或者自己造轮子，手动封装一个自定义指令，这里封装好了一个兼容各浏览器的版本的，主要是判断浏览器支不支持 `IntersectionObserver` API，支持就用它实现懒加载，不支持就用监听 scroll 事件+节流的方式实现

```js
const LazyLoad = {
  // install方法
  install(Vue, options) {
    const defaultSrc = options.default
    Vue.directive("lazy", {
      bind(el, binding) {
        LazyLoad.init(el, binding.value, defaultSrc)
      },
      inserted(el) {
        if (IntersectionObserver) {
          LazyLoad.observe(el)
        } else {
          LazyLoad.listenerScroll(el)
        }
      },
    })
  },
  // 初始化
  init(el, val, def) {
    el.setAttribute("data-src", val)
    el.setAttribute("src", def)
  },
  // 利用IntersectionObserver监听el
  observe(el) {
    var io = new IntersectionObserver((entries) => {
      const realSrc = el.dataset.src
      if (entries[0].isIntersecting) {
        if (realSrc) {
          el.src = realSrc
          el.removeAttribute("data-src")
        }
      }
    })
    io.observe(el)
  },
  // 监听scroll事件
  listenerScroll(el) {
    const handler = LazyLoad.throttle(LazyLoad.load, 300)
    LazyLoad.load(el)
    window.addEventListener("scroll", () => {
      handler(el)
    })
  },
  // 加载真实图片
  load(el) {
    const windowHeight = document.documentElement.clientHeight
    const elTop = el.getBoundingClientRect().top
    const elBtm = el.getBoundingClientRect().bottom
    const realSrc = el.dataset.src
    if (elTop - windowHeight < 0 && elBtm > 0) {
      if (realSrc) {
        el.src = realSrc
        el.removeAttribute("data-src")
      }
    }
  },
  // 节流
  throttle(fn, delay) {
    let timer
    let prevTime
    return function (...args) {
      const currTime = Date.now()
      const context = this
      if (!prevTime) prevTime = currTime
      clearTimeout(timer)

      if (currTime - prevTime > delay) {
        prevTime = currTime
        fn.apply(context, args)
        clearTimeout(timer)
        return
      }

      timer = setTimeout(function () {
        prevTime = Date.now()
        timer = null
        fn.apply(context, args)
      }, delay)
    }
  },
}
export default LazyLoad
```

使用上是这样的，用 `v-LazyLoad` 代替 `src`

```html
<img v-LazyLoad="xxx.jpg" />
```

## 13. SSR

这一点我在项目中也没有实践过，就不班门弄斧了，关于 SSR 的优化可以看这篇文章： [Vue-SSR 优化方案详细总结](https://zhuanlan.zhihu.com/p/93199714)，这里记录一下，就不搬过来了
