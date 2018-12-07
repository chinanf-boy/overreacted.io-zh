---
title: 为啥 React (元素)Elements 有一个 $$typeof 属性(Property)?
date: '2018-12-03'
spoiler: 这有关于安全的东东.
---

你可能认为你在写 JSX:

```jsx
<marquee bgcolor="#ffa7c4">hi</marquee>
```

但实际上,你在调用一个函数:

```js
React.createElement(
  /* type */ 'marquee',
  /* props */ { bgcolor: '#ffa7c4' },
  /* children */ 'hi'
)
```

这个函数还给你一个对象。我们称这个对象为"React"_元素_。 它告诉 React 接下来要渲染什么。你组件返回一棵关于它们的(虚拟)树，像这样。

```js{9}
{
  type: 'marquee',
  props: {
    bgcolor: '#ffa7c4',
    children: 'hi',
  },
  key: null,
  ref: null,
  $$typeof: Symbol.for('react.element'), // 🧐 这谁啊
}
```

如果你使用了 React，你可能对 `type`,`props`,`key`和`ref`字段感到熟悉。**但是`$$typeof`是什么? 为什么它有一个`Symbol()`作为值?**

这是另一件你学习运用 React，不*需要* 知道的事情，但这样地深入会让你感觉很好(feel good)。这篇文章有一些你可能想知道的安全提示。也许有一天你会编写自己的 UI 库，所有的这些都会派上用场。我衷心希望如此.

---

在客户端 UI 库变得常用，和添加基本保护之前，应用程序代码通常会构造 HTML ，并将其插入 DOM:

```js
const messageEl = document.getElementById('message')
messageEl.innerHTML = '<p>' + message.text + '</p>'
```

这也不错，除了当你的`message.text`变得有点像`'<img src onerror="stealYourPassword()">'`的时候，**你不会想让陌生人写的东西在你的应用程序的 HTML 中逐字出现的.**

(有趣的事实:如果你只做客户端渲染，这里的`<script>`标签是不允许你运行 JavaScript。但是[不要 让 这些个东东](https://gomakethings.com/preventing-cross-site-scripting-attacks-when-using-innerhtml-in-vanilla-javascript/)带你陷入一种虚假的安全感.

为了防止此类攻击，可以使用类似 `document.createTextNode()`或`textContent`只处理文本的 安全API。还可以通过替换潜在危险字符`<`,`>`以及其他用户提供的文本，来抢占"逃脱字符"的输入。

尽管如此,错误的代价仍然很高，并且每次在输出中，插入用户编写的字符串时都有可能忘记它。**这就是为什么现代框架，默认情况下喜欢为字符串做转义文本内容的原因:**

```js
<p>{message.text}</p>
```

如果`message.text`是一个恶意字符串，带有`<img>`或者另一个标签，它不会变成真实的`<img>`标签。React 会逃脱内容(转义文本)，*然后*将其插入 DOM。因此,看不到`<img>`标签，你只会看到它的标记。

若要在 React 元素内呈现任意 HTML，必须编写`dangerouslySetInnerHTML={{ __html: message.text }}`。 **写得这么笨拙的事实，这是一个*功能*.** (长的)高度可见，以便您可以在代码审查和代码基础审计中捕获它。

---

**这是否意味着 React 完全不受注入攻击的影响? 不。** HTML 和 DOM 有太多[充足的攻击目标](https://github.com/facebook/react/issues/3473#issuecomment-90594748)，而这要处理起来，对 React 或其他 UI 库来说太困难或太慢。剩余的大多数攻击向量，会涉及属性。例如，如果渲染`<a href={user.website}>`，当心网站的用户填入`'javascript: stealYourPassword()'`。 还有扩展(语法糖)用户输入`<div {...userData}>`挺少见的，但也很危险。

React[能](https://github.com/facebook/react/issues/10506)随着时间的推移提供更多的保护，但在很多情况下的结果,[应该](https://github.com/facebook/react/issues/3473#issuecomment-91327040)是服务器问题，而这无论如何都要修复。

尽管如此,转义文本内容，是捕捉大量潜在攻击的第一道合格(合理)防线。知道下面这样的代码是安全的，难道不好吗?

```js
// 自动转义
<p>{message.text}</p>
```

**嗯,这也不总是正确的。**，而这时就需要`$$typeof`介入了.

---

React 元素设计的原始对象:

```js
{
  type: 'marquee',
  props: {
    bgcolor: '#ffa7c4',
    children: 'hi',
  },
  key: null,
  ref: null,
  $$typeof: Symbol.for('react.element'),
}
```

通常你用`React.createElement()`创建它们，当然这不是唯一途径。有一些有效的用例来支持我刚才写的纯元素对象。当然,你可能不会*希望*像这样写 - 但这[能够](https://github.com/facebook/react/pull/3583#issuecomment-90296667)对于优化编译器、在(内部)工作人员之间传递 UI 元素或从 React 包中解耦 JSX 来说，非常有用。

然而,**如果服务器有一个漏洞,用户可以存储任意的 JSON 对象**，时机正好时客户端代码期望一个字符串，这可能成为一个问题:

```js{2-10,15}
// 服务器有一个漏洞,用户存储 JSON
let expectedTextButGotJSON = {
  type: 'div',
  props: {
    dangerouslySetInnerHTML: {
      __html: '/* put your exploit(破坏吧，尽情地破坏吧) here */',
    },
  },
  // ...
}
let message = { text: expectedTextButGotJSON }

//  React 0.13 的 危险
;<p>{message.text}</p>
```

在这种情况下,React 0.13 是一个防御 XSS 攻击的[弱鸡](http://danlec.com/blog/xss-via-a-spoofed-react-element)。再次澄清**此攻击依赖于现有服务器漏洞.**。 不过, 至 React 0.14 开始，React 可以更好地保护人们免受它的伤害

React 0.14 的修正是[每个 React 元素，标记(tag) 一个 Symbol](https://github.com/facebook/react/pull/4832):

```js{9}
{
  type: 'marquee',
  props: {
    bgcolor: '#ffa7c4',
    children: 'hi',
  },
  key: null,
  ref: null,
  $$typeof: Symbol.for('react.element'),
}
```

这是因为你不能在 JSON 中放`Symbol`。**因此,即使服务器有安全漏洞并返回 JSON ，而不是文本，还有 JSON 也没有包括`Symbol.for('react.element')`。** React 会检查`element.$$typeof`如果元素丢失或无效,将拒绝处理该元素。

使用`Symbol.for()`的好处，具体来说就是**Symbols 是 iframes 和 workers 的 全局变量。**因此,这个修复程序不会阻止。应用程序的不同部分之间传递受信任的元素，即使在更奇怪的情况下。类似地,即使页面上有多个 React 副本，它们仍会"同意"有效`$$typeof`值.

---

那，那些[不支持](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol#Browser_compatibility)Symbols 的浏览器 呢?

唉,他们没有得到额外的保护。React 仍会在元素内，包含`$$typeof`字段，为了一致性，但它是[设为一个 number](https://github.com/facebook/react/blob/8482cbe22d1a421b73db602e1f470c632b09f693/packages/shared/ReactSymbols.js#L14-L16)-`0xeac7`.

为什么，是这个数字，有啥特别? 只是`0xeac7`看起来有点像"React".
