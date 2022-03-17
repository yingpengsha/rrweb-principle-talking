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

# RRWeb 页面录制功能原理分享

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
  - 序列化
  - 增量快照
  - 回放
  - 沙盒
  - 高精度计时器
- 实战
  - 示例演示
  - 插件系统
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

# 页面的序列化和反序列化

也可以称之为**页面快照的生成和复现**

<v-click>

### 方案一：复制 DOM

```js
// record
const snapshot = $("body").clone();
// replay
$("body").replaceWith(snapshot);
```

#### 缺陷

- 快照的数据类型无法作为网络信息传递内容传递

</v-click>

<v-click>

### 方案二：记录 HTML

```js
// record
const snapshot = $("body").innerHTML;
// replay
$("body").innerHTML = snapshot;
```

#### 缺陷

- 会遗漏一些重要的信息，如 `input` 内用户输入的内容

</v-click>

---

# 自定义序列化

也可以理解为使用自定义 DSL 去记录页面内容

### 相对 HTML 的特殊处理
<br>

- **去脚本化** - 被录制页面中的所有 JavaScript 都不应该被执行。
- **记录没有反映在 HTML 中的视图状态** - 例如 `<input type="text" />` 输入后的值不会反映在其 HTML 中，而是通过 value 属性记录。
- **相对路径转换为绝对路径** - 因为回放时的环境 99% 与录制时不同。
- **尽量记录 CSS 样式表的内容** - 将外联样式表转换为内联样式表，防止回放环境导致样式崩塌。

是否应该让更多的外部资源变成内联资源？

---
layout: two-cols
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

# 增量快照
