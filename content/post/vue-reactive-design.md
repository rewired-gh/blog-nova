+++
author = 'Rewired'
date = '2024-03-21'
title = '形象分析 Vue 3 响应式的原理与思路'
tags = [
  'Web 兔子洞',
  'Vue'
]
menu = "main"
+++

## 前情提要

大厂面试很喜欢考「Vue 原理」云云的东西，时不时还会问「是否阅读过 Vue 源码之类的问题」。因此，网上已经有很多类似「Vue 源码导读」之类的资源，然而往往只是给源码片段分别做一点点解说，而并没有从系统层面去分析设计动机和逻辑。本文主要是笔者对于 Vue 3 响应式原理的一些拙见，希望相比于所谓的「Vue 源码导读」能提供更多的见解。

## 基础响应式

### 第一印象

不妨假设有如下的需求：

> `obj` 是一个对象，拥有变量 `obj.v`，我们想用变量 `v_successor` 表示 `obj.v + 1`，且无论 `obj.v` 如何变化都保持这个关系。

需要注意的是，我们不想让 `v_successor` 作为一个函数，因为在更复杂的依赖关系中，函数无法解决这个问题。要解决这个问题，一个显然的思路是：每当 `obj.v` 发生变化时，执行 `v_successor = obj.v + 1`。在 Vue 中等效的代码如下：

```
const obj = reactive({v: 0})
let v_succesor
effect(() => {v_successor = obj.v + 1})
```

此处，`effect` 创建的时候会立即调用传入的闭包（不妨称之为 `fn`）一次。问题来了，`fn` 是在什么时候被注册监听的呢？是如何被知道的呢？如果只阅读源码或者一些文档的话，可能这个过程并没有那么显然。因此，此处先介绍最基础的流程：

1. 调用 `effect` 时，会返回一个将 `fn` 包裹的函数（不妨称之为 `ReactiveEffect`），并且立即执行 `ReactiveEffect`。
2. 执行 `ReactiveEffect` 时，会重新计算依赖，并确保全局定义的 `activeEffect` 被赋值为自己。（这个描述对实际情况有所简化，后文可能会展开讨论）
3. 调用 `fn`。
4. `fn` 中读取了一个响应式对象（不妨称之为 `UnwrapNestedRef`）的属性，因此被通过 Proxy API（请参阅 [Modern JavaScript Tutorial](https://javascript.info/proxy)）的 get 钩子捕获到了。
5. 在 get 钩子中，获悉到是 `activeEffect` 调用自己的，因此记住 `activeEffect` 指向的 `ReactiveEffect`，以便在以后自己触发 set 钩子时调用它。(track 过程)

然后，如果我们此时考虑 `obj.v++` 操作，则会有以下的流程：

1. `obj` 这个 `Proxy` 的 set 钩子捕获到此操作，先原封不动地执行操作。
2. 然后，`obj` 遍历被记住且依赖自己的 `ReactiveEffect`，并且逐个调用。（trigger 过程）
3. 执行 `ReactiveEffect` 后发生的流程请参照前文。

所以，其实 Vue 的响应式实现并没有什么黑魔法，也不是「只用分析一次依赖关系」这种开销最小化的方案，而是频繁相互沟通并计算依赖的方法。以上所有流程从通俗的角度上来说，就是如下的沟通过程：

1. （创建 `effect`）订阅者告诉所有人：「现在轮到自己执行了，自己是目前唯一的执行者。」
2. 发布者察觉到有人读取自己，而这个人只能是目前唯一的执行者，因此把他记下来，列在监听自己的人的名单上。
3. （对 `obj.v` 赋值）发布者察觉到有人设置自己，于是设置完后，按照名单逐个唤醒监听自己的人。
4. 订阅者被唤醒了，告诉所有人自己是目前唯一的执行者。
5. 发布者又察觉到有人读取自己，结果发现这个人是熟人，已经在监听自己的人的名单上了。

### 实现精简版的 Vue 响应式

```
let activeWatch: (() => void) | null = null

const reactive = <T extends Object>(obj: T) => {
  const watcherMap: Map<String | Symbol, Set<() => void>> = new Map()
  const handler: ProxyHandler<T> = {
    get(target, property, receiver) {
      let watchers = watcherMap.get(property)
      // 当有观察者观察且该观察者尚未被注册
      if (activeWatch && !watchers?.has(activeWatch)) {
        if (!watchers) {
          watchers = new Set()
          watcherMap.set(property, watchers)
        }
        // 注册正在观察自己的观察者
        watchers.add(activeWatch)
      }
      return Reflect.get(target, property, receiver)
    },
    set(target, property, value, receiver) {
      // 先设置
      const returnVal = Reflect.set(target, property, value, receiver)
      let watchers = watcherMap.get(property)
      if (watchers) {
        watchers.forEach((handler) => {
          // 调用观察者
          handler()
        })
      }
      return returnVal
    }
  }
  return new Proxy(obj, handler);
}

const effect = <T extends unknown> (fn: () => T, callback?: () => void, lazy: boolean = false) => {
  let wrappedCallback: () => void
  const getWrapped = <U extends unknown> (inner: () => U) => {
    return () => {
      // 告知潜在的响应式对象目前的观察者的回调函数
      activeWatch = wrappedCallback
      // 调用传入函数
      const returnVal = inner()
      // 观察完成
      activeWatch = null
      return returnVal
    }
  }
  const wrappedFn = getWrapped(fn)
  wrappedCallback = callback ? getWrapped(callback) : wrappedFn
  if (!lazy) {
    // 立刻执行一遍以在响应式对象中注册自己
    wrappedFn()
  }
  return wrappedFn
}

const computed = <T extends unknown>(getter: () => T) => {
  let dirty = true
  let value: T
  const runner =  effect(getter, () => {
    // 仅标记数据脏而不立刻重新计算
    dirty = true
  }, true)
  return {
    get value() {
      if (dirty) {
        // 当数据脏时才在读取时重新计算
        value = runner()
        dirty = false
      }
      return value
    }
  }
}

// 例子
const objA = reactive({ a: 1 })
const objB = reactive({ b: 1 })

// follower 随 obj.a 的变化而变化
const follower = computed(() => objA.a + objB.b)

// 更改了响应式对象的属性
objA.a = 2, objB.b = 2
// 期望输出：4
console.log(follower.value)

// 更改了响应式对象的属性
objA.a = 3, objB.b = 3
// 期望输出：6
console.log(follower.value)
```

## 更多内容

本文尚未完成，欢迎订阅本博客的 [RSS](/index.xml)，方便在第一时间获悉更新。