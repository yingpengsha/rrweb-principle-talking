---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://source.unsplash.com/collection/94734566/1920x1080
# apply any windi css classes to the current slide
class: "text-center"
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# persist drawings in exports and build
drawings:
  persist: false
---

# RRWeb 页面录制及回放实现分享

<div class="pt-12">
  <span @click="$slidev.nav.next" class="px-2 py-1 rounded cursor-pointer" hover="bg-white bg-opacity-10">
    Press Space for next page <carbon:arrow-right class="inline"/>
  </span>
</div>

---

# What is RRWeb?

RRWeb 是 'record and replay the web' 的简写，旨在利用现代浏览器所提供的强大 API 录制并回放任意 web 界面中的用户操作。

<br>
<br>

- 🧑‍💻 **用户行为分析** - 像素级地还原用户在页面上的行为，可以更好的理解用户并优化用户体验。
- 🛠 **缺陷复现** - 可以在保持一致性的情况下异步的复现并调试缺陷，这可以帮助开发者更好的修复缺陷。
- 🎥 **更智慧的录制方式** - RRWeb 并非以视频录制的方式记录页面行为，而是用一种更轻量级的方式做到像素级的记录。
- 🕙 **实时分享** - 借助 RRWeb 的能力，你可以更方便的分享你页面上发生的事情。

<br>
<br>
<br>

<div class="flex gap-5px text-32px">
  <a href="https://github.com/rrweb-io/rrweb" target="_blank">
    <radix-icons:github-logo />
  </a>
  <a href="https://www.rrweb.io/" target="_blank">
    <gis:earth-america-o />
  </a>
</div>

<style>
h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 100%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
}
</style>

---

# Navigation

<br>
<br>

- 原理分析
  - 记录
    - 页面序列化
    - 增量变化记录
  - 回放
- 实战
  - 示例演示
- 总结

---
layout: image-right
image: https://images.unsplash.com/photo-1485846234645-a62644f84728?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=1718&q=80
---

# 引言

<br>

RRWeb 的实现原理可以简单概括为。记录**页面的快照**和**发生在视图上的变化**，并配合其自身的序列化和反序列化规则重现用户在页面上的行为。

<br>

举一个相对恰当的例子 🌰，页面上的元素是**演员和道具**，一段时间内页面上发生的一系列试图变化则是**剧本**。只要演员能够严苛的按照剧本演出，这场话剧就能在任何时间、任何地点进行表演并保持一样的效果。

---

# 页面快照及序列化

<v-click>

### 方案一：复制 DOM

<br>

<div class="flex gap-20px items-center">
<div class="flex-1">

```js
// record
const snapshot = $("body").clone();
// replay
$("body").replaceWith(snapshot);
```

</div>

<div class="flex-1">

- **缺陷** - 快照的数据类型无法在网络传递

</div>
</div>

</v-click>

<v-click>

<br>

### 方案二：记录 HTML

<br>

<div class="flex gap-20px items-center">
<div class="flex-1">

```js
// record
const snapshot = $("body").innerHTML;
// replay
$("body").innerHTML = snapshot;
```

</div>

<div class="flex-1">

- **缺陷** - 会遗漏一些重要的信息，如 `input` 内用户输入的内容

</div>
</div>

</v-click>

---

# 自定义序列化

使用自定义 DSL 去记录页面内容

### 相对 HTML 的特殊处理
<br>

- **去脚本化** - 被录制页面中的所有 JavaScript 都不应该被执行。
- **记录没有反映在 HTML 中的视图状态** - 例如 `<input type="text" />` 输入后的值不会反映在其 HTML 中，而是通过 value 属性记录。
- **相对路径转换为绝对路径** - 因为回放时的环境 99% 与录制时不同。
- **尽量记录 CSS 样式表的内容** - 将外联样式表转换为内联样式表，防止回放环境导致样式崩塌。

是否应该让更多的外部资源变成内联资源？

---
layout: two-cols
clicks: 6
---

<div class="pr-10px">

```html {all|1,5|2,4|3|3|3|all} {at:0}
<html>
<body>
  <input type="text" />
</body>
</html>
```

</div>

::right::

<div class="pl-10px">

```json {all|2-5|7-10|12-19|17|21-23|all} {at:0}
{
  "id": 1,
  "type": "Element",
  "tagName": "html",
  "attributes": {},
  "childNodes": [{
    "id": 2,
    "type": "Element",
    "tagName": "body",
    "attributes": {},
    "childNodes": [{
      "id": 3,
      "type": "Element",
      "tagName": "input",
      "attributes": {
        "type": "text",
        "value": "input string"
      },
      "childNodes": []
    },{ 
      "id": 4, 
      "type": "Text",
      "textContent": "\n    "
    }
...
```

</div>

---

# 增量变化记录

前文提到，RRWeb 的基本策略是记录 “快照” + “变化行为”，然后复现整个页面的过程，而这一块要分享的就是 RRWeb 是如何记录页面变化的。

<v-click>

<div class="text-14px">

- DOM 变动
  - 节点创建、销毁
  - 节点属性变化
  - 文本变化
- 鼠标移动
- 鼠标交互
  - mouse up、mouse down
  - click、double click、context menu
  - focus、blur
  - touch start、touch move、touch end
- 页面或元素滚动
- 视窗大小改变
- 输入
- ...canvas/iframe/WebGL

</div>

</v-click>

---

# DOM 变动

<v-click>

[MutationObserver](https://developer.mozilla.org/zh-CN/docs/Web/API/MutationObserver) 接口提供了监视对DOM树所做更改的能力。它被设计为旧的Mutation Events功能的替代品，该功能是DOM3 Events规范的一部分。

<v-click>

## 注意点

**MutationObserver** 的触发方式是**批量且异步**的，这就会导致我们试图记录一些视图变化行为的时候可能会有歧义，同时也有了一系列优化的空间。RRWeb 专门对这些歧义做了细致地处理。

1. 新增节点
   1. 防止重复新增 DOM，需要对每个新增行为对应的 DOM 的子孙节点进行校验。
2. 删除节点
   1. 当新增 DOM 和删除 DOM 的 Mutation 在一个批次中的时候需要进行先后顺序的记录。
3. 属性的高频变化优化
   1. 只取其一个批次变化中的终态

</v-click>

</v-click>

---

# 输入行为

<div v-click-hide="2">

<v-click>

### 人为输入

对 `<input>`, `<textarea>`, `<select>` 三种元素的输入行为进行监听。

</v-click>

</div>

<v-after at="2">

<div style="margin-top: -89px">

```ts
export function hookSetter<T>(
  target: T,
  key: string | number | symbol,
  d: PropertyDescriptor,
  isRevoked?: boolean,
  win = window,
): hookResetter {
  const original = win.Object.getOwnPropertyDescriptor(target, key);
  win.Object.defineProperty(
    target,
    key,
    isRevoked ? d: {
      set(value) {
        // put hooked setter into event loop to avoid of set latency
        setTimeout(() => {
          d.set!.call(this, value);
        }, 0);
        if (original && original.set) {
          original.set.call(this, value);
        }
      },
    },
  );
  return () => hookSetter(target, key, original || {}, true);
}
```

</div>

</v-after>

---

# 其他内容

- Canvas
- WebGL
- IFrame

---

# 回放

RRWeb 默认使用 iframe 来渲染回放的内容

### 一些难点及解决方案

<br>

- 高精度计时器
  <!-- - `requestAnimationFrame` -->
- `hover` 的模拟
  <!-- - 无法直接用程序触发 `hover` 伪类
  - 将 `hover` 相关的伪类样式转成 `.hover` 的 class 样式
  - 使用 `mouseup` 和 `mousedown` 判断是否 `hover`，并添加对应 class -->
- 跨域资源访问问题

---

# 实战

---

# 总结

- [statcounter](https://statcounter.com/)
- [Sentry](https://sentry.io/welcome/)
- 产品星球
