---
layout:     post
title:      "Vue 学习笔记"
subtitle:   ""
date:       2018-4-15 09:00
author:     "Yezh"
header-img: "img/vue.png"
header-mask:  true
catalog:      true
published:    true
tags:
    - JS
    - vue
---

## 介绍

Vue.js（读音 /vjuː/, 类似于 view） 是一套构建用户界面的渐进式JavaScript框架。  
Vue 只关注视图层， 采用自底向上增量开发的设计。  
Vue 的目标是通过尽可能简单的 API 实现响应的**数据绑定**和组合的**视图组件**。  

---

## 安装

兼容性: 支持所有兼容 ECMAScript 5 的浏览器, 不支持 IE8 及以下版本。

最新稳定版本: 2.5.16 (截至2018年4月)

每个版本的更新日志见 [vue_releases](https://github.com/vuejs/vue/releases)

### 引入script

对于Vue.js的使用, 我们可以通过直接下载对应版本并用 \<script\> 标签引入，Vue 会被注册为一个全局变量。

下面给出了两种不同的版本:    
[开发版本](https://vuejs.org/js/vue.js): 包含完整的警告和调试模式  
[生产版本](https://vuejs.org/js/vue.min.js): 压缩版本, 删除了警告

另外, 我们还可以通过[CDN](https://www.zhihu.com/question/37353035), 直接在标签中输入对应URL来链接到指定版本的Vue.js, 例如:  
```html
<script src="https://cdn.jsdelivr.net/npm/vue@2.5.16/dist/vue.js"></script>
```

### Vue Devtools
vue-devtools 是一款由vue官方编写的用于调试vue应用的调试工具，该工具可以有效提高调试效率。

此处我是直接通过 chrome应用商店 下载安装的。

手动安装流程可参考  [vue-devtools-github](https://github.com/vuejs/vue-devtools#vue-devtools)
或者 [Vue调试神器vue-devtools安装](https://segmentfault.com/a/1190000009682735)

注: 在使用前需要确认是否已在扩展程序页面的 `Vue.js.devtools` 下勾选 `允许访问文件网址` 选项, 没有的话是无法对本地文件使用的。

---

## 第一个Vue应用: Hello Vue!

```html
<!-- helloVue.html -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Hello Vue!</title>
    <script src="https://cdn.jsdelivr.net/npm/vue@2.5.16/dist/vue.js"></script>    
</head>
<body>
    <div id="app">
        {\{ message }\}
    </div>
    
    <script type="text/javascript" src="./helloVue.js"></script>
</body>
</html>
```
注: 上面的`{\{ message }\}` 实际上应该去掉 `\` 但是去掉后jekyll会默认其为变量, 并尝试替代, 而由于并不存在该变量, 会导致无任何输出。所以加上 `\` 只是为了使其能够显示。

```js
// helloVue.js
var app = new Vue({
    el: '#app',
    data: {
        message: "Hello Vue!"
    }
})
```

用浏览器打开该HTML文件后可以看到网页上输出了 "Hello Vue!" 文本。

---

## 常用指令
Vue指令以v-开头，作用在HTML元素上(可将其视作特殊的HTML标签属性), 可以给绑定的元素添加一些特殊行为, 从而扩展HTML功能。  


### v-bind

v-bind指令会将所绑定的元素节点的指定属性与Vue实例中的某个属性保持一致。  
语法为: `v-bind:attribute="property"`

例子:
```html
<div id="app-2">
  <span v-bind:title="message">
    鼠标悬停几秒钟查看此处动态绑定的提示信息！
  </span>
</div>
```

```js
var app2 = new Vue({
  el: '#app-2',
  data: {
    message: '页面加载于 ' + new Date().toLocaleString()
  }
})
```

### v-if

条件判断指令，根据表达式值的真假来插入或删除元素。  
语法为： `v-if = "expression"`

例子:
```html
<!-- html -->
<div id="app-3">
  <p v-if="seen">现在你看到我了</p>
</div>
```

```js
// JS
var app3 = new Vue({
  el: '#app-3',
  data: {
    seen: true
  }
})
```

### v-for

循环指令，用于遍历数组、对象等，与JavaScript遍历类似。  
语法为：`v-for = "item in items"`

例子:
```html
<!-- html -->
<div id="app-4">
  <ol>
    <li v-for="todo in todos">
      {{ todo.text }}
    </li>
  </ol>
</div>
```

```js
// JS
var app4 = new Vue({
  el: '#app-4',
  data: {
    todos: [
      { text: '学习 JavaScript' },
      { text: '学习 Vue' },
    ]
  }
})
```

### v-on

v-on指令为绑定的元素节点添加一个事件监听器, 以监听DOM事件。  
语法为: `v-on:event = "doSomeThing"`

例子:
```html
<!-- html -->
<div id="app-5">
  <p>{\{ message }\}</p>
  <button v-on:click="reverseMessage">逆转消息</button>
</div>
```

```js
// JS
var app5 = new Vue({
  el: '#app-5',
  data: {
    message: 'Hello Vue.js!'
  },
  methods: {
    reverseMessage: function () {
      this.message = this.message.split('').reverse().join('')
    }
  }
})
```

### v-model

v-model指令实现了双向数据绑定, 一般用于表单输入和应用状态之间的双向绑定。  
语法为: `v-model="property"`

例子:
```html
<!-- html -->
<div id="app-6">
  <p>{{ message }}</p>
  <input v-model="message">
</div>
```

```js
// JS
var app6 = new Vue({
  el: '#app-6',
  data: {
    message: 'Hello Vue!'
  }
})
```

> 注:  
    1. 以上例子均来自于Vue官网教程;  
    2. 该笔记将会随着学习进度持续更新。


