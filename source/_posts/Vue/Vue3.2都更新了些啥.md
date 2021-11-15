---
title: Vue3.2都更新了些啥
date: 2020-10-14 00:03:07
author: 沐华
img: https://cdn.jsdelivr.net/gh/wmuhua/cdn@main/blog/24.jpg
top: false
cover: false
coverImg: /images/1.jpg
password:
toc: false
mathjax: false
summary: 对于Vue3.2，尤大说：“真香，没有理由不用”。一起来看看都有哪些变化吧
categories: Vue
tags:
  - Vue
  - Vue3.2
---

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eedb514ab22a4631adc2887d1ef34d97~tplv-k3u1fbpfcp-watermark.image)

8 月 10 号尤大宣布 Vue3.2 发布，尤大发布的英文版说明原文[点这里](https://blog.vuejs.org/posts/vue-3.2.html)，[github 点这里](https://github.com/vuejs/vue-next/blob/master/CHANGELOG.md)

根据原文内容的更新的内容主要有以下 5 块：

- `SSR`：服务端渲染优化。`@vue/server-renderer`包加了一个 ES 模块创建，与 Node.js 解耦，使在非 Node 环境用`@vue/serve-render`做服务端渲染成为可能，比如(Workers、Service Workers)
- `New SFC Features`：新的单文件组件特性
- `Web Components`：自定义 web 组件。这个我们平时很少用到，但是应该知道
- `Effect Scope API`： effect 作用域，用来直接控制响应式副作用的释放时间(computed 和 watchers)。这是底层库的更新，开发不用关心，但是应该知道
- `Performance Improvements`：性能提升。这是内部的提升，跟开发无关

emmmm.... 原文基本就是这样，说的不清不楚的，这一块放到后面吧

先介绍一下开发相关的变更

## 新特性和 API

### 新的 SFC 单文件组件特性

可以在 `style` 标签里使用 `v-bind`，如下

```js
<template>
    <div>{{ color }}</div>
    <button @click="color = color === 'red' ? 'green' : 'red'">按钮</button>
</template>
<script setup>
    import { ref } from "vue"
    const color = ref("ref")
</script>
<style scoped>
    div{
        color: v-bind(color)
    }
</style>
```

### 废弃 useContext

原来的 `useContext` 是这样的，现在下面这个都不能用了

```js
import { useContext } from "vue"
const { expose, slots, emit, attrs } = useContext()
```

由于 `expose`、`slots`、`emit`、`attrs` 都不能通过 `useContext` 获取了，随之而来的是下面几个新的代替方式

### 新增 useAttrs、useSlots

不能通过 useContext 获取了，就像下面这样获取

```js
import { useAttrs, useSlots } from "vue"
const attrs = useAttrs()
const slots = useSlots()
```

### 新增 defineExpose

不需要通过 import 导入，向下面这样直接使用，功能一样，对外暴露属性和方法

```js
defineExpose({
    name:"沐华"
    someMethod(){
        console.log("这是子组件的方法")
    }
})
```

### defineEmit 改名

原来是可以向上面那样通过 useContext() 获取或者向下面这样导入的

```js
import { defineEmit } from "vue"
```

改名后，加了一个 `s` ，并且`不需要通过 import 导入`，像下面这样直接使用

```js
// import { defineEmit } from "vue"
defineEmits(["getName", "myClick"])
```

### defineProps 变更

本来是需要通过 import 导入的，变更后不需要导入，直接像下面这样使用

```js
defineProps(["name"])
//或者
defineProps({
    name: String
})
//或者
defineProps({
    name: {
        type: String,
        default: "",
        ...
    }
})
//或者
const props = defineProps({
    ... // 和上面一样
})
console.log(props.name)
```

### 顶级 await

不需要再用 `async` 就可以直接使用 `await`，这样默认会把组件的 `setup` 变为 `async setup`，像下面这样

```js
<script setup lang="ts">
  const post = await fetch(`/api/post/xxx`).then(res => res.json())
</script>
```

最终会转换成下面这样

```js
<script lang="ts">
    import { defineComponent, withAsyncContext } from "vue"
    export default defineComponent({
        async setup(){
            const post = await withAsyncContext(
                fetch(`/api/post/xxx`).then(res => res.json())
            )
            return {
                post
            }
        }
    })
</script>
```

### 新增 withDefaults

在 `TS` 中，像下面这样定义 `props` 是不能设置默认值的

```js
interface Props{
    name: string,
    age: number
}
defineProps<Props>()
```

加入 `withDefaults` 之后就可以指定默认值，像下面这样

```js
import { withDefaults } from
interface Props{
    name: string,
    age: number
}
const props = withDefaults(defineProps<Props>(), {
    name: "沐华"
    age: 3
})
```

### 自定义 web 组件

通过 `defineCustomElement` 方法创建原生自定义组件。也可以通过这种方式发布组件出去**卖**给别人使用

```js
// main.js
import { defineCustomElement } from "vue"

const MyVueElement = defineCustomElement({
  // 通用 vue 组件选项
  props: ["foo"],
  render() {
    return h("div", "my-vue-element:" + this.foo)
  },
  // 仅适用于 defineCustomElement, css将被注入到 shadow root
  style: [`div { border: 1px solid red }`],
})

customElements.define("my-vue-element", MyVueElement)
```

然后在 `vite.config.js` 里配置白名单

```js
export default defineConfig({
  plugins: [
    vue({
      template: {
        compilerOptions: {
          // vue 将跳过 my-vue-element 解析
          isCustomElement: (tag) => tag === "my-vue-element",
        },
      },
    }),
  ],
})
```

然后使用

```js
<my-vue-element foo="foo" />
```

## 新增指令 v-memo

官方说的是我们写程序很少有场景能用到。[官方文档点这里](https://v3.vuejs.org/api/directives.html#v-once)

它可以**缓存模板中的一部分**，从而提升速度。比如说大量 v-for 的列表，只创建一次，就不会再更新了，直接用缓存，就是用内存换时间

下面这样组件重新渲染时，如果 valueA 和 valueB 没有变化，div 将跳过此组件和其子组件的所有更新

```js
<div v-memo="[valueA, valueB]">...</div>
```

还有像下面这样，部分缓存。需要注意的是在 `v-memo 里面不能用 v-for`

```js
<div v-for="item in list" :key="item.id" v-memo="[item.id === selected]">
    ...
</div>
```

## 性能提升

响应式的优化

- 更高效的 `ref` 实现，读取提升约 `260%`，写入提升约 `50%`
- 依赖收集速度提升约 `40%`
- 减少内存消耗约 `17%`

模板编译器优化

- 创建元素 VNodes 速度提升约 `200%`

看这数据是真的吊
