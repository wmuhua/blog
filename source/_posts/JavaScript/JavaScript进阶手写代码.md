---
title: JavaScript进阶手写代码
date: 2020-06-12 22:39:06
author: 沐华
img: https://cdn.jsdelivr.net/gh/wmuhua/cdn@main/blog/15.jpg
top: false
cover: true
coverImg: /images/1.jpg
password:
toc: false
mathjax: false
summary: 本文主要是记录开发中一些常用的方法手写实现
categories: JavaScript
tags:
  - JavaScript
  - 手写代码
---

本文代码比较多，大部分说明都在注释里了

## 解析 url 参数

就是提出 url 里的参数并转成对象

```js
function getUrlParams(url) {
  let reg = /([^?&=]+)=([^?&=]+)/g
  let obj = {}
  url.replace(reg, function () {
    obj[arguments[1]] = arguments[2]
  })
  return obj
}
let url = "https://www.junjin.cn?a=1&b=2"
console.log(getUrlParams(url)) // { a: 1, b: 2 }
```

## call

改变 this 指向用的，可以接收多个参数

```js
Function.prototype.myCall = function (ctx) {
  ctx = ctx || window // ctx 就是 obj
  let fn = Symbol()
  ctx[fn] = this // this 就是 foo
  let result = ctx[fn](...arguments)
  delete ctx[fn]
  return result
}
let obj = { name: 沐华 }
function foo() {
  return this.name
}
// 就是把 foo 函数里的 this 指向，指向 obj
console.log(foo.myCall(obj)) // 沐华
```

用 `Symbol` 是因为他是独一无二的，避免和 obj 里的属性重名

原理就是把 foo 添加到 obj 里，执行 foo 拿到返回值，再从 obj 里把 foo 删掉

## apply

原理同上，只不过 apply 接收第二个参数是数组，不支持第三个参数

```js
Function.prototype.myApply = function (ctx) {
  ctx = ctx || window
  let fn = Symbol()
  ctx[fn] = this
  let result
  if (arguments[1]) {
    result = ctx[fn](...arguments[1])
  } else {
    result = ctx[fn]()
  }
  delete ctx[fn]
  return result
}
```

## bind

```js
Function.prototype.myBind = function (ctx, ...args) {
  const self = this
  const fn = function () {}
  const bind = function () {
    const _this = this instanceof fn ? this : ctx
    return self.apply(_this, [...args, ...arguments])
  }
  fn.prototype = this.prototype
  bind.prototype = new fn()
  return bind
}
```

bind 不会立即执行，会返回一个函数

- 函数可以直接执行并且传参，如 `foo.myBind(obj, 1)(2, 3)`，所以需要 `[ ...args, ...arguments ]`合并参数
- 函数也可以 `new`，所以要判断原型 `this instanceof fn`

然后实现原型继承，如果对原型不太了解的话，请移步我上一篇文章 [助力进击大厂，JavaScript 前端考点总结](https://juejin.cn/post/6995591618371780622)

**call、apply、bind 的区别**

- 都可以改变 `this` 指向
- call 和 apply 会`立即执行`，bind 不会，而是返回一个函数
- call 和 bind 可以接收`多个参数`，`apply` 只能接受两个，第二个是`数组`
- bind 参数可以分多次传入

## instanceof

说明在注释里，接受两个参数，判断第二个参数是不是在第一个参数的原型链上

```js
function myInstanceof(left, right) {
  // 获得实例对象的原型 也就是 left.__proto__
  let left = Object.getPrototypeOf(left)
  // 获得构造函数的原型
  let prototype = right.prototype
  // 判断构造函数的原型 是不是 在实例的原型链上
  while (true) {
    // 原型链一层层向上找，都没找到 最终会为 null
    if (left === null) return false
    if (prototype === left) return true
    // 没找到就把上一层拿过来，继续循环，再向上一层找
    left = Object.getPrototypeOf(left)
  }
}
```

## 数组去重

个人感觉这个还蛮喜欢考的

```js
// 来个示例数组
let arr = [
  1,
  1,
  "1",
  "1",
  true,
  true,
  "true",
  {},
  {},
  "{}",
  null,
  null,
  undefined,
  undefined,
]

// 方法一
let unique1 = Array.from(new Set(arr))
console.log(unique1) // [1, "1", true, "true", {}, {}, "{}", null, undefined]

// 方法二
let unique2 = (arr) => {
  let map = new Map() // 或者用空对象 let obj =｛｝利用对象属性不能重复的特性
  let brr = []
  arr.forEach((item) => {
    if (!map.has(item)) {
      // 如果是对象的话就判断 !obj[item]
      map.set(item, true) // 如果是对象的话就 obj[item] = true  其他一样
      brr.push(item)
    }
  })
  return brr
}
console.log(unique2(arr)) // [1, "1", true, "true", {}, {}, "{}", null, undefined]

// 方法三
let unique3 = (arr) => {
  let brr = []
  arr.forEach((item) => {
    // 使用 indexOf  返回数组是否包含某个值 没有就返回 -1 有就返回下标
    if (brr.indexOf(item) === -1) brr.push(item)
    // 或者使用 includes 返回数组是否包含某个值  没有就返回false  有就返回true
    if (!brr.includes(item)) brr.push(item)
  })
  return brr
}
console.log(unique3(arr)) // [1, "1", true, "true", {}, {}, "{}", null, undefined]

// 方法四
let unique4 = (arr) => {
  // 使用 filter 返回符合条件的集合
  let brr = arr.filter((item, index) => {
    return arr.indexOf(item) === index
  })
  return brr
}
console.log(unique4(arr)) // [1, "1", true, "true", {}, {}, "{}", null, undefined]
```

上面的方法不能对引用类型去重，除非指针一样，指针是可以去重的，比如下面这样是可以去重的

```js
let crr = []
let arr = [crr, crr]
```

## 数组扁平化

就是把多维数组变成一维数组

```js
// 来个示例数组
let arr = [1, [2, [3, [4, [5]]]]]

// 方法一
// flat() 默认拉平一层嵌套数组，传入数字几就拉平几层
// Infinity 是无穷大，不管嵌套多少层都给你拉平
let brr1 = arr.flat(Infinity)
console.log(brr1) // [1, 2, 3, 4, 5]

// 方法二
// 转成字符串，再去掉字符串里的 “[” 和 “]”，再把字符串转回数组
let brr2 = JSON.parse("[" + JSON.stringify(arr).replace(/\[|\]/g, "") + "]")
console.log(brr2) // [1, 2, 3, 4, 5]

// 方法三
let brr3 = (arr) => {
  // 用递归，用 for 循环加递归也可以，这里用 reduce
  // reduce 累计器，本质上也是循环，
  // cur 是循环的当前一个值，相当于 for循环里的arr[i]， pre 是前一个值，相当于for循环里的arr[i-1]
  let crr = arr.reduce((pre, cur) => {
    return pre.concat(Array.isArray(cur) ? brr3(cur) : cur)
  }, [])
  return crr
}
console.log(brr3(arr)) // [1, 2, 3, 4, 5]
```

## 防抖

连续点击的情况下不会执行，只在最后一下点击过指定的秒数后才会执行

应用场景：点击按钮，输入框模糊查询，词语联想等

```js
function debounce(fn, wait) {
  let timeout = null
  return function () {
    // 每一次点击判断有延迟执行的任务就停止
    if (timeout !== null) clearTimeout(timeout)
    // 否则就开启延迟任务
    timeout = setTimeout(fn, wait)
  }
}
function sayDebounce() {
  console.log("防抖成功！")
}
btn.addEventListener("click", debounce(sayDebounce, 1000))
```

## 节流

频繁触发的时候，比如滚动或连续点击，在指定的间隔时间内，只会执行一次

应用场景：点击按钮，监听滚动条，懒加载等

```js
// 方案1  连续点击的话，每过 wait 秒执行一次
function throttle(fn, wait) {
  let bool = true
  return function () {
    if (!bool) return
    bool = false
    setTimeout(() => {
      // fn() // fn中this指向window
      fn.call(this, arguments) // fn中this指向btn  下面同理
      btn = true
    }, wait)
  }
}
// 方案2 连续点击的话，第一下点击会立即执行一次 然后每过 wait 秒执行一次
function throttle(fn, wait) {
  let date = Date.now()
  return function () {
    let now = Date.now()
    // 用当前时间 减去 上一次点击的时间 和 传进来的时间作对比
    if (now - date > wait) {
      fn.call(this, arguments)
      date = now
    }
  }
}
function sayThrottle() {
  console.log("节流成功！")
}
btn.addEventListener("click", throttle(sayThrottle, 1000))
```

## new

```js
function myNew(fn, ...args) {
  // 不是函数不能 new
  if (typeof fn !== "function") {
    throw new Error("TypeError")
  }
  // 创建一个继承 fn 原型的对象
  const newObj = Object.create(fn.prototype)
  // 将 fn 的 this 绑定给新对象，并继承其属性，然后获取返回结果
  const result = fn.apply(newObj, args)
  // 根据 result 对象的类型决定返回结果
  return result && (typeof result === "object" || typeof result == "function")
    ? result
    : newObj
}
```

## Object.create

```js
function createObj(obj) {
  function Fn() {}
  Fn.prototype = obj
  return new Fn()
}
```

创建一个空对象并修改原型，这没啥说的，一般传个 null 进去，这样创建出来没有原型的对象不会被原型污染，或者传要继承的对象原型

## Es5 继承

```js
// 创建一个父类
function Parent(){}
Parent.prototype.getName = function(){ return '沐华' }
// 子类
function Child(){}

// 方式一
Child.prototype = Object.create(Parent.prototype)
Child.prototype.constructor = Child // 重新指定 constructor
// 方式二
Child.prototype = Object.create(Parent.prototype，{
  constructor:{
    value: Child,
    writable: true, // 属性能不能修改
    enumerable: true, // 属性能不能枚举(可遍历性)，比如在 for in/Object.keys/JSON.stringify
    configurable: true, // 属性能不能修改属性描述对象和能否删除
  }
})

console.log(new Child().getName) // 沐华
```

ES5 的继承方式有很多种，什么原型链继承、组合继承、寄生式继承...等等，了解一种面试就够用了

## Es6 继承

```js
// 创建一个父类
class Parent(){
  constructor(props){
    this.name = '沐华'
  }
}
// 创建一个继承自父类的子类
class Child extends Parent{
  // props是继承过来的属性， myAttr是自己的属性
  constructor(props, myAttr){
    // 调用父类的构造函数，相当于获得父类的this指向
    super(props)
  }
}
console.log(new Child().name) // 沐华
```

## 深拷贝

```js
// 用 while 写一个通用的遍历
function myEach(array, iteratee) {
  let index = -1
  const length = array.length
  while (++index < length) {
    iteratee(array[index], index)
  }
  return array
}
function myClone(target, map = new WeakMap()) {
  // 引用类型才继续深拷贝
  if (target instanceof Object) {
    const isArray = Array.isArray(target)
    // 克隆对象和数组类型
    let cloneTarget = isArray ? [] : {}

    // 防止循环引用
    if (map.get(target)) {
      // 有拷贝记录就直接返回
      return map.get(target)
    }
    // 没有就存储拷贝记录
    map.set(target, cloneTarget)

    // 是对象就拿出同级的键集合  返回是数组格式
    const keys = isArray ? undefined : Object.keys(target)
    // value是对象的key或者数组的值 key是下标
    myEach(keys || target, (value, key) => {
      if (keys) {
        // 是对象就把下标换成value
        key = value
      }
      // 递归
      cloneTarget[key] = myClone(target[key], map)
    })
    return cloneTarget
  } else {
    return target
  }
}
```

## 获取数据类型

```js
function getType(value) {
  if (value === null) {
    return value + ""
  }
  if (typeof value === "object") {
    // 数组、对象、null 用 typeof 都是 object，所以需要处理下 以 {} 为例
    let valueClass = Object.prototype.toString.call(value) // 转成这样 [object, Object]
    let type = valueClass.split(" ")[1].split("") // 变成这样 ["O", "b", "j", "e", "c", "t", "]"]
    type.pop() // 再变成这样 ["O", "b", "j", "e", "c", "t"]
    return type.join("").toLowerCase() // object
  } else {
    return typeof value
  }
}
console.log(getType(1)) // number
console.log(getType("1")) // string
console.log(getType(null)) // null
console.log(getType(undefined)) // undefined
console.log(getType({})) // object
console.log(getType(function () {})) // function
```

## 函数柯里化

柯里化函数就是高阶函数的一种，好处主要是实现参数的复用和延迟执行，不过性能上就会没那么好，要创建数组存参数，要创建闭包，而且存取 argements 比存取命名参数要慢一点

实现 `add(1)(2)(3)` 要求参数不固定，类似 `add(1)(2, 3, 4)(5)()` 这样也行，我这实现的是中间的不能不传参数，最后一个不传参数，以此来区分是最后一次调用然后累计结果

```js
// 每次调用的传进来的参数做累计处理
function reduce(...args) {
  return args.reduce((a, b) => a + b)
}
function currying(fn) {
  // 存放每次调用的参数
  let args = []
  return function temp(...newArgs) {
    if (newArgs.length) {
      // 有参数就合并进去，然后返回自身
      args = [...args, ...newArgs]
      return temp
    } else {
      // 没有参数了，也就是最后一个了，执行累计结果操作并返回结果
      let val = fn.apply(this, args)
      args = [] //保证再次调用时清空
      return val
    }
  }
}
let add = currying(reduce)
console.log(add(1)(2, 3, 4)(5)()) //15
console.log(add(1)(2, 3)(4, 5)()) //15
```

## AJAX

```js
// 这是调用下面函数的结构，这个结果大家应该很熟悉，就不解释了，应该都用过
myAjax({
  type: "get",
  url: "https://xxx",
  data: { name: "沐华", age: 18 },
  dataType: "json",
  async: true,
  success: function (data) {
    console.log(data)
  },
  error: function () {
    alert("报错")
  },
})
// 定义一个将 { name: "沐华", age:18 } 转成 name=沐华&age=18 这种格式的方法
function fn(data) {
  let arr = []
  for (let i in data) {
    arr.push(i + "=" + data[i])
  }
  return arr.join("&")
}
// 下面就是实现上面调用和传参的函数
function myAjax(options) {
  let xhr = null
  let str = fn(options.data)
  // 创建 xhr
  if (window.XMLHttpRequest) {
    xhr = new XMLHttpRequest()
  } else {
    xhr = new ActiveXObject("Microsoft,XMLHTTP")
  }
  // 这里只配置了 get 和 post
  if (options.type === "get" && options.data !== undefined) {
    // 创建 http 请求
    xhr.open(options.type, options.url + "?" + str, options.async || true)
    // 发送请求
    xhr.send(null)
  } else if (options.type === "post" && options.data !== undefined) {
    xhr.open(options.type, options.url, options.async || true)
    // 设置请求头
    xhr.setRequestHeaders("Content-type", "application/x-www-form-urlencoede")
    xhr.send(str)
  } else {
    xhr.open(options.type, options.url, options.async || true)
    xhr.send(null)
  }
  // 监听状态
  xhr.onreadystatechange = function () {
    if (xhr.readyState === 4 && xhr.status === 200) {
      let res = xhr.responseText
      try {
        if (options.success === undefined) {
          return xhr.responseText
        } else if (typeof res === "object") {
          options.success(res)
        } else if (options.dataType === "json") {
          options.success(JSON.parse(res))
        } else {
          throw new Error()
        }
      } catch (e) {
        if (options.error !== undefined) {
          options.error()
          throw new Error()
        } else {
          throw new Error()
        }
      }
    }
  }
}
```

## Promise

```js
class MyPromise {
  constructor(fn) {
    // 存储 reslove 回调函数列表
    this.callbacks = []
    const resolve = (value) => {
      this.data = value // 返回值给后面的 .then
      while (this.callbacks.length) {
        let cb = this.callbacks.shift()
        cb(value)
      }
    }
    fn(resolve)
  }
  then(onResolvedCallback) {
    return new MyPromise((resolve) => {
      this.callbacks.push(() => {
        const res = onResolvedCallback(this.data)
        if (res instanceof MyPromise) {
          res.then(resolve)
        } else {
          resolve(res)
        }
      })
    })
  }
}
// 这是测试案例
new MyPromise((resolve) => {
  setTimeout(() => {
    resolve(1)
  }, 1000)
})
  .then((res) => {
    console.log(res)
    return new MyPromise((resolve) => {
      setTimeout(() => {
        resolve(2)
      }, 1000)
    })
  })
  .then((res) => {
    console.log(res)
  })
```

完整的 Promise 实在太长了，比 AJAX 还要长很多很多，所以就实现个极简版的，只有 `resolve` 和 `then` 方法，可以无限 `.then`

## Promise.all

Promise.all 可以把多个 Promise 实例打包成一个新的 Promise 实例。传进去一个值为多个 Promise 对象的数组，成功的时候返回一个结果的数组，返回值的顺序和传进去的顺序是一致对应得上的，如果失败的话就返回最先 reject 状态的值

如果遇到需要同时发送多个请求并且按顺序返回结果的话，Promise.all 就可以完美解决这个问题

```js
MyPromise.all = function (promisesList) {
  let arr = []
  return new MyPromise((resolve, reject) => {
    if (!promisesList.length) resolve([])
    // 直接循环同时执行传进来的promise
    for (const promise of promisesList) {
      promise.then((res) => {
        // 保存返回结果
        arr.push(res)
        if (arr.length === promisesList.length) {
          // 执行结束 返回结果集合
          resolve(arr)
        }
      }, reject)
    }
  })
}
```

## Promise.race

传参和上面的 all 一模一样，传入一个 Promise 实例集合的数组，然后全部同时执行，谁先快先执行完就返回谁，只返回一个结果

```js
MyPromise.race = function (promisesList) {
  return new MyPromise((resolve, reject) => {
    // 直接循环同时执行传进来的promise
    for (const promise of promisesList) {
      // 直接返回出去了，所以只有一个，就看哪个快
      promise.then(resolve, reject)
    }
  })
}
```

## 双向数据绑定

```js
let obj = {}
let input = document.getElementById("input")
let box = document.getElementById("box")
// 数据劫持
Object.defineProperty(obj, "text", {
  configurable: true,
  enumerable: true,
  get() {
    // 获取数据就直接拿
    console.log("获取数据了")
  },
  set(newVal) {
    // 修改数据就重新赋值
    console.log("数据更新了")
    input.value = newVal
    box.innerHTML = newVal
  },
})
// 输入监听
input.addEventListener("keyup", function (e) {
  obj.text = e.target.value
})
```

## 路由

```js
// 简易版的 hash 路由
class myRoute {
  constructor() {
    // 路由存储对象
    this.routes = {}
    // 当前hash
    this.currentHash = ""
    // 绑定this，避免监听时this指向改变
    this.freshRoute = this.freshRoute.bind(this)
    // 监听
    window.addEventListener("load", this.freshRoute, false)
    window.addEventListener("hashchange", this.freshRoute, false)
  }
  // 存储
  storeRoute(path, cb) {
    this.routes[path] = cb || function () {}
  }
  // 更新
  freshRoute() {
    this.currentHash = location.hash.slice(1) || "/"
    this.routes[this.currentHash]()
  }
}
```
