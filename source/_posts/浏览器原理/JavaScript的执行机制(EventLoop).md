---
title: JavaScript的执行机制(EventLoop)
date: 2020-04-24 00:49:06
author: 沐华
img: https://cdn.jsdelivr.net/gh/wmuhua/cdn@main/blog/7.jpg
top: true
cover: true
coverImg: /images/1.jpg
password: 
toc: false
mathjax: false
summary: JavaScript执行机制(EventLoop)
categories: 浏览器原理
tags:
  - 浏览器原理
  - JavaScript
---

## 我

上一篇文章介绍了进程与线程，知道渲染进程都有一个主线程，并且主线程工作很多，要处理DOM、计算样式、布局、还有鼠标、键盘等各种JS任务

我们都知道JS是单线程，任务只能一件一件地执行，那么浏览器是怎么让这么多类型的任务在主线程上有条紊地执行的呢？

这就需要**任务队列**和**事件循环**了

## 任务队列(消息队列)

什么是任务队列呢？

它是一种数据结构，存放要执行的任务。然后事件循环系统再以先进先出原则按顺序执行队列中的任务。产生新任务时IO线程就将任务添加在队列尾部，要执行任务渲染主线程就会循环地从队列头部取出执行，如图

![未标题-12.jpg](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4b00d4477e9c4bca8d2268c4ceca8a9b~tplv-k3u1fbpfcp-watermark.image)

如果其他进程也有任务想让主线程执行的话，也是一样的通过 IO 线程接收并将任务添加到任务队列就可以了

可任务队列里的任务类型太多了，而且是多个线程操作同一个任务队列，比如鼠标滚动、点击、移动、输入、计时器、WebSocket、文件读写、解析DOM、计算样式、计算布局、JS执行.....

这些任务都在主线程中执行，而JS是单线程的，一个任务执行需要等前面的任务都执行完，所以就需要解决单个任务占用主线程过久的问题

比如如果一个动画任务前面有一个JS任务执行时间很长，那我们看到的就是卡卡的感觉，用户体验就很不好

如果是DOM频繁发生变化的JS任务，每次变化都需要调用相应的JavaScript接口，无疑会导致任务时间拉长，如果把DOM变化做成异步任务，那可能添加到任务队列过程中，前面又有很多任务在排队了

所以`为了处理高优先级的任务`，`和解决单任务执行过长的问题`，所以需要将任务划分，所以微任务和宏任务它来了

在说微任务之前，要知道一个概念就是**同步**和**异步**

## 同步和异步

我们知道了浏览器页面是由任务队列和事件循环系统来驱动的，但是队列要一个一个执行，如果某个任务(http请求)是个耗时任务，那浏览器总不能一直卡着，所以为了防止主线程阻塞，JavaScript 又分为同步任务和异步任务

`同步任务`：就是任务一个一个执行，如果某个任务执行时间过长，后面的就只能一直等下去

`异步任务`：就是进程在执行某个任务时，该任务需要等一段时间才能返回，这时候就把这个任务放到专门处理异步任务的模块，然后继续往下执行，不会因为这个任务而阻塞

**也就是说，除了任务队列，还有一个专门处理需要延迟执行的模块(延迟哈希表)**

**常见的异步任务**：定时器、ajax、事件绑定、回调函数、async await、promise

好了，我们再来说微任务吧

## 微任务和宏任务

JS执行时，V8会创建一个全局执行上下文，在创建上下文的同时，**V8也会在内部创建一个微任务队列**

**有微任务队列，自然就有宏任务队列，任务队列中的每一个任务则都称为宏任务，在当前宏任务执行过程中，如果有新的微任务产生，就添加到微任务队列中**

- `微任务包括`： promise回调、proxy、MutationObserver(监听DOM)、node 中的 process.nextTick等
- `宏任务包括`： 渲染事件、请求、script、setTimeout、setInterval、Node中的setImmediate、I/O 等

**来看栗子搞懂她**

你和一个大爷在银行办业务，大爷排在你前面，大爷是要存钱，存完钱之后，工作人员问大爷还要不要办理其他业务，大爷说那我再改个密码吧，这时候总不能让大爷到队伍最后去排队再来改密码吧

这里面大爷要办业务就是一个宏任务，而在钱存完了又想改密码，这就产生了一个微任务，大爷还想办其他业务就又产生新微任务，直到所有微任务执行完，队伍的下一个人再来

这个队伍就是任务队列，工作人员就是单线程的JS引擎，排队的人只能一个一个来让他给你办事

也就是说`当前宏任务里的微任务全部执行完，才会执行下一个宏任务`

**用代码来举例**

```js
<script> // 宏任务
    console.log(1)
    setTimeout(()=>{ // 宏任务
        console.log(2)
    },0)
    console.log(3)
</script>
```
    
输出结果就是 `1  3  2` ，因为setTimeout是宏任务，哪怕它的时间为0，当前宏任务里的任务没执行完，她插队也没用。然后就算计时时间为0，它也是一个延迟任务，所以放到异步处理模块去先

**注意**：异步处理模块(延迟哈希表)是一个和任务队列同等级的数据结构。每个宏任务结束后，主线程就会检查延迟哈希表，将里面到期的任务拿出来依次执行，比如回调/计时器达到触发条件等。不明白的话下面有图，看着就很清晰了

**再来**

```js
<script> // 宏任务
    console.log(1)
    new Promise( resolve => {
        resolve(2) // 回调 是微任务
        console.log(3)
    }).then( num => {
        console.log(num)
    })
    console.log(4)
</script>
```

输出结果就是 `1 3 4 2` ，遇到微任务(这里的回调)就放到微任务队列里，等着执行栈中的任务执行完，再拿出来执行

**看图，必须要搞懂她**

![367e4062e66b2c2512768749e533393.jpg](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/25681734818441f7a16ea6cdc0e0bbc3~tplv-k3u1fbpfcp-watermark.image)

没理解的话可以多看一会儿这个图，这一块儿也是面试很爱问的

如图可以看出来执行过程形成了一个循环，这就是**事件循环( EventLoop )**

## 事件循环( `EventLoop` )

事件循环：一句话概括就是入栈到出栈的循环

即：一个宏任务，所有微任务，渲染，一个宏任务，所有微任务，渲染.....

**循环过程**： 

1. 所有同步任务都在主线程上依次执行，形成一个执行栈(调用栈)，异步任务处理完后则放入一个任务队列

2. 当执行栈中任务执行完，再去检查微任务队列里的微任务是否为空，有就执行，如果执行微任务过程中又遇到微任务，就添加到微任务队列末尾继续执行，把微任务全部执行完

3. 微任务执行完后，再到任务队列检查宏任务是否为空，有就取出最先进入队列的宏任务压入执行栈中执行其同步代码

4. 然后回到第2步执行该宏任务中的微任务，如此反复，直到宏任务也执行完，如此循环

## 练习一下，彻底搞懂她

```js
<script>
    setTimeout(function () {
        console.log('setTimeout')
    }, 0)
    new Promise(function (resolve) {
        console.log('promise1')
        for( let i = 0; i < 1000; i++ ) {
            i === 999 && resolve()
        }
        console.log('promise2')
    }).then(function ()  {
        console.log('promise3')
    })
    console.log('script')
</script>
```

输出结果：`promise1` -> `promise2` -> `script` -> `promise3` -> `setTimeout`

**想一下为什么？**

- script 是宏任务，先执行它里面的微任务
- 遇到宏任务setTimeout放到异步处理模块(延迟哈希表)
- 继续执行promise，打印`promise1`
- 遇到循环，执行，遇到回调 resolve()，上面说了回调属于微任务，放到微任务队列
- 继续执行，打印 `promise2`
- 继续执行，打印 `script`
- 执行栈的任务执行完了，去微任务列队里拿
- 有一个 then 回调，执行，打印 `promise3`
- 微任务都执行完了，去任务队列拿下一个宏任务
- 执行 setTimeout，打印 `setTimeout`

没有理解的话，再想想

### 当遇到 async / await 呢？

`async`/`await` 是 ES7 引入的重大改进的地方，**可以在不阻塞主线程的情况下，使用同步代码实现异步访问资源的能力**，让我们的代码逻辑更清晰

说白了

`async`：就是异步执行和隐式返回Promise  
`await`：返回的就是一个Promise对象

**看题**

```js
async function fun() {
    console.log(1)
    let a = await 2
    console.log(a)
    console.log(3)
}
console.log(4)
fun()
console.log(5)
```

输出结果：`4 1 5 2 3`

结合 async / await 的特点，我们来把这个题用 ES6 翻译一下

```js
function fun(){
    return new Promise(() => {
        console.log(1)
        Promise.resolve(2).then( a => {
            console.log(a)
            console.log(3)
        })
    })
}
console.log(4)
fun()
console.log(5)
```

上面说了，回调是微任务，所以直接扔到微任务队列等着，这题里自然就是最后执行，是不是好理解一点了

**再来**

先别看下面答案，想一下这题，和上面有一点点区别

```js
function bar () {
    console.log(2)
}
async function fun() {
    console.log(1)
    await bar()
    console.log(3)
}
console.log(4)
fun()
console.log(5)
```

输出结果：`4 1 2 5 3`

为啥？上面例子中 2 都没打印出来，为啥这个就出来了

因为await的意思就是等，等await后面的执行完。所以"await bar()"，是从右向左执行，执行完bar()，然后遇到await，返回一个微任务(哪怕这任务里没东西)，放到微任务队列让出主线程。

上面说了 async/await 就是把异步以同步的形式实现，同步就是一步一步一行一行来嘛，await在微任务队列里都没回来，那在await下面的自然不能执行，导致 3 最后打印

下面还有一题，把上面几题结合了，我就不写答案了

### 你来搞定她

```js
async function async1 ()  {
    console.log('async1 start');
    await  async2();
    console.log('async1 end')
}

async function  async2 ()  {
    console.log('async2')
}

console.log('script start');

setTimeout(function ()  {
    console.log('setTimeout')
},  0);

async1();

new Promise(function (resolve)  {
    console.log('promise1');
    resolve()
}).then(function ()  {
    console.log('promise2')
});

console.log('script end')
```
