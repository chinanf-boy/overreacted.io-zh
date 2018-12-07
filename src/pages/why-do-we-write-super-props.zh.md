---
title: 为什么我们编写了 super(props)?
date: '2018-11-30'
spoiler: 这为 最后一个 twist(曲解).
---

我听到了[Hooks](https://reactjs.org/docs/hooks-intro.html)是新的热度。具有讽刺意味的是，我想通过描述关于*class*组件的有趣事实，来开始这个博客。好是吧！

**这些东西对你生产使用 React ，都*不*怎么重要。但如果你喜欢深入研究事物是如何运作的,你可能会发现它们的有趣.**

这是第一个.

---

在我的时间里，我写`super(props)`比我知道的，要占更多时间:

```js{3}
class Checkbox extends React.Component {
  constructor(props) {
    super(props)
    this.state = { isOn: true }
  }
  // ...
}
```

当然,[class(类) 字段 提议](https://github.com/tc39/proposal-class-fields)让我们跳过'仪式':

```js
class Checkbox extends React.Component {
  state = { isOn: true }
  // ...
}
```

像这样的语法是[提上计划的](https://reactjs.org/blog/2015/01/27/react-v0.13.0-beta-1.html#es7-property-initializers)，那时是 React 0.13 在 2015 增加了对纯类的支持。定义`constructor`和调用`super(props)`一直是临时解决方案，直到类字段提供了符合人体工程学的替代方案。

但是让我们回到这个例子,只使用 ES2015 特性:

```js{3}
class Checkbox extends React.Component {
  constructor(props) {
    super(props)
    this.state = { isOn: true }
  }
  // ...
}
```

**为什么我们调用`super`? 我们能*不*调用它? 如果我们不得不使用该函数,如果我们不传递`props`,会发生什么? 还有其他参数(arguments)吗?** 让我们来查一下.

---

在 JavaScript 中,`super`引用父类构造函数。(在我们的例子中,它指向`React.Component`的类实现).

重要的是,你不能一开始在构造函数中使用`this`，直到您已经调用父构造函数。JavaScript 让你这样:

```js
class Checkbox extends React.Component {
  constructor(props) {
    // 🔴 还不能使用 `this`
    super(props)
    // ✅ 现在 好了
    this.state = { isOn: true }
  }
  // ...
}
```

有一个很好的理由说明，为什么 JavaScript 需在您触摸`this`之前，强制执行父构造函数。 考虑类层次结构:

```js
class Person {
  constructor(name) {
    this.name = name
  }
}

class PolitePerson extends Person {
  constructor(name) {
    this.greetColleagues() // 🔴 不被允许, 往下看为啥
    super(name)
  }
  greetColleagues() {
    alert('Good morning folks!')
  }
}
```

想象一下，若`super`调用之前，使用`this`*是*允许的。一个月后,我们可能会改变`greetColleagues`中某人的名字的信息:

```js
  greetColleagues() {
    alert('Good morning folks!');
    alert('My name is ' + this.name + ', nice to meet you!');
  }
```

但是我们忘记了，在`super()`有机会初建`this.name`之前，我们就已调用了`this.greetColleagues()`。 所以`this.name`还没有定义呢! 正如你所看到的,这样的代码很难思考.

为了避免这种陷阱,**JavaScript 强制执行,如果您想使用`this`在构造函数中,你*不得不*首先调用`super`.**让父母做自己的事情! 而这个限制也适用于定义为类的 React 组件:

```js
  constructor(props) {
    super(props);
    // ✅ 好了。现在可以使用`this`
    this.state = { isOn: true };
  }
```

这给我们带来了另一个问题:为什么传递`props`?

---

你可能认为传`props`下到`super`是必要的,以便基本`React.Component`构造函数，可以初始化`this.props`:

```js
//  React 内部
class Component {
  constructor(props) {
    this.props = props
    // ...
  }
}
```

其实离真相不远 - 事实上,可以看看[做了啥](https://github.com/facebook/react/blob/1d25aa5787d4e19704c049c3cfa985d3b5190e0d/packages/react/src/ReactBaseClasses.js#L22).

但不知何故,即使你用`super()`，但没有`props`参数,你仍然可在`render`以及其他方法，访问`this.props`。(如果你不相信我,你自己试试!)

*这*又是如何工作? 结果是，在调用*你的*构造函数之后，**React 在实例 也分配了`props`:**

```js
// React内部
const instance = new YourComponent(props)
instance.props = props
```

所以即使你忘记传递`props`到`super()`，React 之后仍会让他们正确。而这是有原因的.

当 React 对类(classes)增加支持时，它不仅仅增加了对 ES6 类的支持。我们的目标是尽可能广泛地支持类抽象。但我们对 ClojureScript, CoffeeScript, ES6, Fable, Scala.js, TypeScript 或其他解决方案在定义组件方面有多成功，并[不清楚](https://reactjs.org/blog/2015/01/27/react-v0.13.0-beta-1.html#other-languages) 。因此,React 是故意不自以为是地决定是否使用`super()`，即使对 ES6 类也这样的.

这是否意味着你可以写`super()`，而不是`super(props)`?

**现有可能不是因为它仍困惑.**，而是，React 确实会稍后分配`this.props` ，在您的构造函数已运行*之后*。但在这个`super`调用和构造函数的结尾*之间*，`this.props`仍然是未定义(undefined)的:

```js{14}
// React内部
class Component {
  constructor(props) {
    this.props = props
    // ...
  }
}

// 你的代码
class Button extends React.Component {
  constructor(props) {
    super() // 😬 我们忘记传递 props
    console.log(props) // ✅ {}
    console.log(this.props) // 😬 undefined
  }
  // ...
}
```

如果*从*构造函数的某种方法中，发生这种情况，那么调试可能更具挑战性。.**这就是为什么我建议永远用`super(props)`传递，即使它不是严格(use strict)的:**

```js
class Button extends React.Component {
  constructor(props) {
    super(props) // ✅ 我们传递props
    console.log(props) // ✅ {}
    console.log(this.props) // ✅ {}
  }
  // ...
}
```

这甚至确保了，在构造函数退出之前，`this.props`的设立.

---

最后有一点，长期 React 的用户可能会好奇的.

您可能注意到,当您在类中使用上下文(Context) API 时(与'遗产'`contextTypes`或是 React 16.6 版本添加的现代`contextType`API 一起使用) `context`作为第二个参数传递给构造函数。

那么我们为什么不写成`super(props, context)`呢? 回答是: 我们可以,但上下文较少使用,所以这个陷阱就没有那么多了.

**随着类字段建议(class fields proposal)，这整个陷阱几乎消失了.**如果没有显式构造函数,所有参数都会自动传递。这就是允许`state = {}`表达式之类，还可以在需要时，加上`this.props`或`this.context`的引用。

用 Hooks,我们甚至没有`super`或`this`. 但这是下回的话题.
