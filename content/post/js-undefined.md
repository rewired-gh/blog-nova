+++
author = 'Rewired'
date = '2024-02-22'
title = 'JavaScript 中的 undefined、不存在、delete'
tags = [
  'Web 兔子洞',
  'JavaScript'
]
menu = "main"
+++

## TL; DR

在 JavaScript 中不能通过将对象的属性设置成 `undefined` 来删除属性。

反直觉的是，有的全局变量可以用 `delete` 来删除，有的属性不能用 `delete` 来删除。换句话说，`var obj = 0` 与 `globalThis.obj = 0` 效果上不等价，即使在 99% 的情景中是一样的。

与此同时：

> 所有属性都是平等的，但是有些属性比其他的属性更平等。

## 启发背景

若获取一个存在的对象中不存在的属性（或者可以视作 key），将会得到 `undefined`。

```
> const obj = {}
undefined
> obj.name
undefined
```

若将变量或者属性赋值为 `undefined`，获取时显然也会得到 `undefined`。那么，是否可以通过将属性赋值为 `undefined` 来删除某个属性呢？

## 初步实践

在 JavaScript 中可以通过 `delete` 来删除一个属性；例如：

```
> const obj = { name: "admin" }
undefined
> delete obj.name
true
> Object.keys(obj)
[]
```

若将 `name` 设置为 `undefined`，则：

```
> obj.name = undefined
undefined
> Object.keys(obj)
[ 'name' ]
```

可见，后面这种方式并不能删除属性。

## 进一步讨论

虽然在很多情况下，如果把该对象视作 Java、C# 等语言中的对象来使用时，这两种做法似乎并没有什么区别。但是，在 JavaScript、Python 等语言中，对象可以用作哈希表，而且在解决一些高级需求的时候会「反射」「改造」对象；这时，`delete` 将不能被赋值为 `undefined` 取代，即使 `delete` 这种语言特性的设计给人一种「狗皮药膏」的感觉。

事实上，「获取不存在的属性得到 undefined」显然是 JavaScript 设计中的一个败笔，不过因为历史遗留原因已经无法改正。更好的做法应该是像 Python 一样抛出异常，并且提供一个方便检查属性是否存在的 API。

更糟糕的是，JavaScript 并没有把局部变量和属性在这方面一视同仁。如果获取不存在的变量，将会抛出异常；例如：

```
> console.log(sakana)
Uncaught ReferenceError: sakana is not defined
```

而且变量也不能通过 `delete` 删除；例如：

```
> const sakana = 0
undefined
> delete sakana
false
> sakana
0
```

进一步，全局变量也不可以被 `delete`，即使全局变量本质上是 `window` 或者 `globalThis` 对象的属性；例如：

```
> var obj = 0
undefined
> globalThis.obj
0
> delete obj
false
> delete globalThis.obj
false
```

但是，如果通过往 `window` 或 `globalThis` 添加属性来声明全局变量，则可以被 `delete` 删除；例如：

```
> globalThis.obj = 0
0
> obj
0
> delete obj
true
```

那么，如果我们用 `var` 声明一个 `window` 或 `globalThis` 中已经存在的全局变量呢？如果按照之前发现的规律，这个全局变量应该是不能被删除的。但是，现实真的如此吗？

```
> globalThis.fetch
[Function: fetch]
> var fetch = 1
undefined
> globalThis.fetch
1
> delete fetch
true
> globalThis.fetch
undefined
```

无语凝噎。

## 陈词滥调

在 ES5 以及更早的版本中，`undefined` 是一个变量，可以被更改。因此，如果开发者确实需要用到 `undefined` 的时候，会采用 [`void 0`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/void) 这个写法。此外，由于 `void 0` 相比 `undefined` 占用更少的字符，所以很多代码最小化工具会将 `undefined` 最小化为 `void 0`。



## 结语

不愧是 JavaScript，到处都是狗皮药膏和不一致性的表现，新特性为了兼容性只能为以前的不良设计擦屁股。