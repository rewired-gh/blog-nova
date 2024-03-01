+++
author = 'Rewired'
date = '2024-03-01'
title = '利用 animate 特性制作精美的次级指针特效'
tags = [
  'Web 兔子洞',
  'UI/UX'
]
menu = "main"
+++

## 效果展示

1. 访问 <https://rewired.moe>。
2. 使用鼠标、触控屏等任意指针输入设备，在此页面上尝试点按、拖拽。

## 前置知识

- PointerEvent：<https://developer.mozilla.org/en-US/docs/Web/API/PointerEvent>
- Animation（Baseline 2022）：<https://developer.mozilla.org/en-US/docs/Web/API/Animation>

## 如何实现

### 外观与排版

首先，在 HTML（此处使用 Pug 语言）中添加一个 `span` 用于表示次级指针：

```
span#click-effect
```

需要注意的是，最好将该元素作为 `body` 的 child。

然后，在 CSS（此处使用 SASS 语言）中添加类似如下的样式：

```
#click-effect
  position: fixed // 指针永远在可见区，故不用 absolute
  top: 50% // 锚点居中
  left: 50% // 锚点居中
  z-index: 100 // 保持指针在顶层
  transform: translate(-50%, -50%) // 中心点居中
  height: 0 // 初始为隐藏状态
  aspect-ratio: 1 / 1 // 高度与宽度一致
  border-radius: 50% // 构造圆形
  box-sizing: border-box // 内描边，后续动画使用
  filter: blur(1px)
  border: solid 0 hotpink // 初始无描边
  pointer-events: none // 令指针事件穿透
```

### 动画

#### 进入动画

动画部分采用 DOM 操作实现，使用了在 Baseline 2022 中的 Animation API。若要兼容旧浏览器，请考虑增加额外的判断语句，或者基于旧特性 polyfill。

一开始，可以通过 `querySelector` 获得先前定义的 `span` 的 DOM 对象：

```
const effect = document.querySelector("#click-effect")
```

我们先构造进入动画。进入动画包括「指针从上一个位置迅速移动到当前按下的位置」、「指针变大」、「不透明度逐渐增加」。需要注意的是，我们这里会使用边框来「填充」指针，这个特性后续会用到。初步代码如下：

```
const {left: fromX, top: fromY} = effect.getBoundingClientRect()
const {clientX: toX, clientY: toY} = event
const sliding = {
  left: [`${fromX}px`, `${toX}px`],
  top: [`${fromY}px`, `${toY}px`],
  height: ["0", "16px"],
  opacity: ["0", "20%"],
  borderWidth: ["0", "8px"],
}
const timing = {
  duration: 200,
  fill: "forwards",
}
effect.animate(sliding, timing)
```

为了让指针更具有生命力，我们想让指针「弹一下再收回」。具体改动后的代码如下：

```
const {left: fromX, top: fromY} = effect.getBoundingClientRect()
const {clientX: toX, clientY: toY} = event
const sliding = {
  left: [`${fromX}px`, `${toX}px`, `${toX}px`],
  top: [`${fromY}px`, `${toY}px`, `${toY}px`],
  height: ["0", "18px", "16px"],
  opacity: ["0", "20%", "20%"],
  borderWidth: ["0", "9px", "8px"],
  offset: [0, 0.8],
}
const timing = {
  duration: 240,
  fill: "forwards",
}
effect.animate(sliding, timing)
```

值得注意的是，这里使用了 `forwards` 时间轴特性，使动画结束后元素保持在结束时的状态。不过，这一特性会在后续带来一点麻烦。

然后，将上述代码在 `document` 的 `pointerdown` 事件触发时执行。需要注意的是，「指针事件」相比 `mousedown` 等事件更普适，可以兼容触摸屏、数位板等多种设备。

```
document.addEventListener(
  "pointerdown",
  (event) => {
    // 此处省略了进入动画代码
  }
)
```

#### 退出动画

退出对应着 `pointerup` 和 `pointercancel` 两个事件。因此，在 `pointerdown` 触发时，我们需要注册这两个事件对应的监听器。与此同时，别忘了在退出时取消已经注册的监听器。

```
document.addEventListener(
  "pointerdown",
  (event) => {
    const playPopEffect = (_) => {
      // 此处省略了退出动画代码
      document.removeEventListener("pointerup", playPopEffect)
      document.removeEventListener("pointercancel", playPopEffect)
    }
    document.addEventListener("pointerup", playPopEffect)
    document.addEventListener("pointercancel", playPopEffect)
    // 此处省略了进入动画代码
  }
)
```

退出动画包括「指针变大」、「不透明度减少」、「边框变细直到消失」。因为此前使用边框作为「填充」，所以也可以理解为「填充通过变大的圆形蒙版逐渐消失」。具体代码如下：

```
const popping = {
height: ["16px", "32px"],
opacity: ["20%", "0"],
borderWidth: ["8px", "0"],
}
const timing = {
duration: 400,
fill: "forwards",
}
effect.animate(popping, timing)
```

#### 拖拽时跟随

当用户拖拽指针时，我们想让次级指针跟随拖拽动作。这一效果可以通过监听 `pointermove` 事件实现。一种较为直观的做法如下：

```
const followCursor = (event) => {
  const {clientX: toX, clientY: toY} = event
  effect.style.left = `${clientX}px`
  effect.style.top = `${clientY}px`
}
document.addEventListener("pointermove", followCursor)
```

同时，别忘了在退出动画结束后注销这个监听器。

```
const playPopEffect = (_) => {
  // 此处省略了先前介绍的代码
  document.removeEventListener("pointermove", followCursor)
}
```

然而，这种做法实际上是不奏效的，次级指针并不会跟随拖拽。这是因为进入动画的 `forwards` 时间轴特性，导致元素会一直保持动画所述的样式，即使更改了 inline style。所以，我们只能使用一个无过渡的动画来实现拖拽时跟随。

```
const followCursor = (event) => {
  const {clientX: toX, clientY: toY} = event
  const sliding = {
      left: [`${toX}px`, `${toX}px`],
      top: [`${toY}px`, `${toY}px`],
  }
  const timing = {
      duration: 0,
      fill: "forwards",
  }
  effect.animate(sliding, timing)
}
```

到此为止，我们实现了完整的次级指针特效。

#### 完整代码

前文提到的所有 JavaScript 代码，整合后如下：

```
const effect = document.querySelector("#click-effect")

document.addEventListener(
  "pointerdown",
  (event) => {
    const followCursor = (event) => {
      const {clientX: toX, clientY: toY} = event
      const sliding = {
        left: [`${toX}px`, `${toX}px`],
        top: [`${toY}px`, `${toY}px`],
      }
      const timing = {
        duration: 0,
        fill: "forwards",
      }
      effect.animate(sliding, timing)
    }
    document.addEventListener("pointermove", followCursor)
    const playPopEffect = (_) => {
      const popping = {
        height: ["16px", "32px"],
        opacity: ["20%", "0"],
        borderWidth: ["8px", "0"],
      }
      const timing = {
        duration: 400,
        fill: "forwards",
      }
      effect.animate(popping, timing)
      document.removeEventListener("pointermove", followCursor)
      document.removeEventListener("pointerup", playPopEffect)
      document.removeEventListener("pointercancel", playPopEffect)
    }
    document.addEventListener("pointerup", playPopEffect)
    document.addEventListener("pointercancel", playPopEffect)

    const {left: fromX, top: fromY} = effect.getBoundingClientRect()
    const {clientX: toX, clientY: toY} = event
    const sliding = {
      left: [`${fromX}px`, `${toX}px`, `${toX}px`],
      top: [`${fromY}px`, `${toY}px`, `${toY}px`],
      height: ["0", "18px", "16px"],
      opacity: ["0", "20%", "20%"],
      borderWidth: ["0", "9px", "8px"],
      offset: [0, 0.8],
    }
    const timing = {
      duration: 240,
      fill: "forwards",
    }
    effect.animate(sliding, timing)
  }
)
```

## 讨论

相比于基于 CSS `transition` 的动画特效，利用 `animate` 能实现更丰富和精细化控制的动画，也更方便设计者构思。