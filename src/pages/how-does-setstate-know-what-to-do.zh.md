---
title: `setstate` 是如何知道它应该做什么?
date: '2018-12-09'
spoiler: 如果您不必考虑它，依赖注入是很好的.
---

在一个组件中，调用`setState`，您觉得会发生什么?

```jsx{11}
import React from 'react'
import ReactDOM from 'react-dom'

class Button extends React.Component {
  constructor(props) {
    super(props)
    this.state = { clicked: false }
    this.handleClick = this.handleClick.bind(this)
  }
  handleClick() {
    this.setState({ clicked: true })
  }
  render() {
    if (this.state.clicked) {
      return <h1>Thanks</h1>
    }
    return <button onClick={this.handleClick}>Click me!</button>
  }
}

ReactDOM.render(<Button />, document.getElementById('container'))
```

当然是，React 用下一个`{ clicked: true }` state 重新渲染组件，并更新 DOM 以匹配返回的`<h1>Thanks</h1>`元件.

似乎很简单。但等等，是*React*做的呢?还是*React DOM*?

更新 DOM 听起来就像 React DOM 负责的部分。但我们正在调用`this.setState()`，而不是来自 React DOM 的东西。而我们的`React。Component`基类是在 React 本身内部定义的。

`React.Component`内部的`setState()`怎么才可以更新 DOM?

**免责声明:就像在这个博客上，[更多](/why-do-react-elements-have-typeof-property/) [其他](/how-does-react-tell-a-class-from-a-function/) [博文](/why-do-we-write-super-props/)，你实际上并不*需要*知道这些，就可以高效使用 React 。这篇文章适合那些喜欢看幕后有什么的人。随你选!**

---

我们可能会认为`React.Component`类 含有 DOM 更新的逻辑.

但如果是这样，`this.setState()`怎么才能在其他环境中工作呢? 例如，React Native 应用程序中的组件也会扩展`React.Component`。他们会执行`this.setState()`就像我们上面所做的那样，但 React Native 是使用 Android 和 iOS 本机视图，可不是 DOM。

您可能也熟悉 React Test Renderer 或 Shallow Renderer。这两种测试策略都可以让您呈现正常的组件和会在其中调用`this.setState()`。但它们都不适用于 DOM.

如果您使用的是像[React ART](https://github.com/facebook/react/tree/master/packages/react-art)这样的渲染器，您也许知道可以在页面上使用多个渲染器。(例如，ART 组件能在 React DOM 树中工作。)这使得全局标志或变量无法维持。

所以， **`React.Component`委托了，处理特定平台的状态更新代码，** 而它是怎么样滴？ 不过，在我们了解这是如何发生之前，让我们深入了解包的分离方式和原因.

---

人们普遍存在一种误解，即 React"引擎"存在于内部`react`包。NO!!!

事实上，从[包 分离: React 0.14 版本](https://reactjs.org/blog/2015/07/03/react-v0.14-beta-1.html#two-packages)起,`react`有意地，用来为*定义*组件 才公开的 API。大部分的 React*实现* 生活在"渲染器(renderers)"中.

`react-dom`,`react-dom/server`,`react-native`,`react-test-renderer`,`react-art`是渲染器的一些例子(你可以[自己耍耍](https://github.com/facebook/react/blob/master/packages/react-reconciler/README.md#practical-examples)).

这就是为什么，无论您针对哪个平台，`react`包都能起作用。它的所有出口,如`React.Component`,`React.createElement`,`React.Children`工具集和(必杀技)[Hooks](https://reactjs.org/docs/hooks-intro.html)，都独立于目标平台。无论您运行 React DOM，React DOM Server 还是 React Native，您的组件都会以相同的方式导入和使用它们。

相比之下，渲染器包公开特定平台的 API，如`ReactDOM.render()`能使您将 React 层次结构 **mount** 到 DOM 节点中。每个渲染器都提供像这样的 API。理想情况下，大多数*组件*不应该从渲染器导入任何东西。这使它们更具便携性。

**大多数人都认为 React"引擎"在每个独立的渲染器中。**许多渲染器，都包含相同代码的副本 - 我们称之为[“reconciler”](https://github.com/facebook/react/tree/master/packages/react-reconciler)。这有个[构建 步骤](https://reactjs.org/blog/2017/12/15/improving-the-repository-infrastructure.html#migrating-to-google-closure-compiler)将协调程序(reconciler)代码与渲染器代码一起，绑到一个高度优化的捆绑包中，以获得更好的性能。(直接复制代码对捆绑包大小来说，不能说是好方式，但绝大多数 React 用户一次 **只** 需要 **一个** 渲染器，例如`react-dom`。)

换句话说，`react`包只会让你专于*使用*React 功能，但并不需要知道他们如何实现的任何事情。但你看到这里，也知道了渲染器包(`react-dom`,`react-native`等)提供 React 功能和特定平台的逻辑实现。其中一些代码是共享的("协调程序(reconciler)")，但各个渲染器都有其实现的细节。

---

现在我们知道为什么，`react`和`react-dom`包*都* 需要更新，才能获取新功能的原因了。例如，当 React 16.3 添加了 Context API 时，`React.createContext()`在 React 包上公开。

但`React.createContext()`实际上，并没有*实现*上下文(Context)功能。而,React DOM 和 React DOM Server 之间的实现也需要有所不同。所以`createContext()`会返回一些普通对象:

```js
// 微简化
function createContext(defaultValue) {
  let context = {
    _currentValue: defaultValue,
    Provider: null,
    Consumer: null,
  }
  context.Provider = {
    $$typeof: Symbol.for('react.provider'),
    _context: context,
  }
  context.Consumer = {
    $$typeof: Symbol.for('react.context'),
    _context: context,
  }
  return context
}
```

当在代码中，你使用`<MyContext.Provider>`或`<MyContext.Consumer>`，是*渲染器*决定如何处理它们。React DOM 可能以一种方式跟踪 Context 值，但 React DOM Server 又可能会采用不同的方式。

**所以，若你更新`react`到 16.3+，但不更新`react-dom`，你将使用一个尚未了解特殊`Provider`和`Consumer`类型的渲染器。**这就是为什么年纪大了的`react-dom`会[说这些类型无效，因为它老了啊](https://stackoverflow.com/a/49677020/458193).

同样的警告适用于 React Native。但是，与 React DOM 不同，React 版本不会立即"强制"使用 React Native 版本。他们有独立的发布时间表。更新的渲染器代码是在几周内，一次[单独同步](https://github.com/facebook/react-native/commits/master/Libraries/Renderer/oss)到 React Native 存储库。这就是为什么，功能仅在 React Native 可用，皆因 与 React DOM 计划表不同.

---

好的，现在我们知道了`react`包中没有任何有趣的东西，并且实现代码是生活在渲染器中，如`react-dom`，`react-native`，等等。但这并没有回答我们的问题。`React.Component`内的`setState()`如何与正确的渲染器"交谈"?

**答案是：在创建的类上，每个渲染器设置一个特殊字段。**该字段叫`updater`。这不是*你*自个设置的 - 而是在创建类的实例后，就会立即在 React DOM，React DOM Server 或 React Native 自行设置的:

```js{4,9,14}
// Inside React DOM
const inst = new YourComponent()
inst.props = props
inst.updater = ReactDOMUpdater

// Inside React DOM Server
const inst = new YourComponent()
inst.props = props
inst.updater = ReactDOMServerUpdater

// Inside React Native
const inst = new YourComponent()
inst.props = props
inst.updater = ReactNativeUpdater
```

看看[`React.Component`中的`setState` 实现](https://github.com/facebook/react/blob/ce43a8cd07c355647922480977b46713bd51883e/packages/react/src/ReactBaseClasses.js#L58-L67)，它所做的就是委托，创建此组件实例的渲染器，去工作:

```js
// 微简化
setState(partialState, callback) {
  // 使用`updater`字段 跟 the renderer 回话!
  this.updater.enqueueSetState(this, partialState, callback);
}
```

React DOM Server[也行想](https://github.com/facebook/react/blob/ce43a8cd07c355647922480977b46713bd51883e/packages/react-dom/src/server/ReactPartialRenderer.js#L442-L448)忽略状态更新，并警告你，而 React DOM 和 React Native 会让他们的协调程序(reconciler)副本[处理它](https://github.com/facebook/react/blob/ce43a8cd07c355647922480977b46713bd51883e/packages/react-reconciler/src/ReactFiberClassComponent.js#L190-L207)。

这就是`this.setState()`，即使它在 React 包中定义，也可以更新 DOM 的原因。它读取由 React DOM 设置而来的`this.updater`，再让 React DOM 安排，并处理更新。

---

我们现在了解这堂课的目的啦，你又知道 Hooks 吗?

当人们第一眼看到[Hooks 提案 API](https://reactjs.org/docs/hooks-intro.html)的时候，他们经常想: `useState`怎么"知道要做什么"? 臆测它会比基类`React.Component`的`this.setState()`来的更"神奇"。

但，正如我们今天所见，基类中的`setState()`实现始终是一种幻想。除了将调用转发给当前渲染器之外，它不会执行任何操作，和`useState`钩子[是完全一样的行为](https://github.com/facebook/react/blob/ce43a8cd07c355647922480977b46713bd51883e/packages/react/src/ReactHooks.js#L55-L56).

**这次，就不再是`updater`字段，Hooks 使用"dispatcher"对象。**当你使用`React.useState()`,`React.useEffect()`或者另一个内置的 Hook，这些调用被转发到当前的 dispatcher。

```js
// 在 React (简化)
const React = {
  // 真 property 会藏得深点, 看看你是否能找到它!
  __currentDispatcher: null,

  useState(initialState) {
    return React.__currentDispatcher.useState(initialState)
  },

  useEffect(initialState) {
    return React.__currentDispatcher.useEffect(initialState)
  },
  // ...
}
```

并且各个渲染器在渲染组件之前，都会设置 dispatcher:

```js{3,8-9}
// In React DOM
const prevDispatcher = React.__currentDispatcher
React.__currentDispatcher = ReactDOMDispatcher
let result
try {
  result = YourComponent(props)
} finally {
  // Restore it back
  React.__currentDispatcher = prevDispatcher
}
```

给个示例，React DOM Server 实现是[在这里](https://github.com/facebook/react/blob/ce43a8cd07c355647922480977b46713bd51883e/packages/react-dom/src/server/ReactPartialRendererHooks.js#L340-L354)，以及 React DOM 和 React Native 共享的 reconciler 实现[在这里](https://github.com/facebook/react/blob/ce43a8cd07c355647922480977b46713bd51883e/packages/react-reconciler/src/ReactFiberHooks.js).

这就是为什么，像`react-dom`这样的渲染器需要在你调用 Hooks 包时，访问相同的`react`。不然的话，您的组件将不会"看到" dispatcher! 这也可能导致，当你在同一个组件树中，使用[多个 React 副本](https://github.com/facebook/react/issues/13991)不起作用。但是，这总是引发些模糊的错误，所以 Hook 会强迫您在付费/填坑之前，解决包重复问题。

虽然我们不鼓励以下做法，但在技术上，您完全可以自行覆盖 dispatcher ，以获得高级工具用例。(我谎称了`__currentDispatcher`名称，但您可以在 React 仓库中找到真的那个。) 例如，React DevTools 将使用[一个 专用的 dispatcher](https://github.com/facebook/react/blob/ce43a8cd07c355647922480977b46713bd51883e/packages/react-debug-tools/src/ReactDebugHooks.js#L203-L214)，通过捕获 JavaScript 堆栈跟踪来反思 Hooks 树。_不要在家里重复这个（Don’t repeat this at home）。_

这也意味着 Hooks 本身并不依赖于 React。如果将来有更多的库，想要重用相同的原始 Hook，理论上 dispatcher 可以移动到一个单独的包中，并以一些不"可怕"名称的第一类 API 公开。在实践中，我们直到需要它的那一刻，也不会避免过早的抽象。

这俩啊，`updater`字段和`__currentDispatcher`对象是通用编程原理的形式，被称为*依赖注入*。在这两种情况下，渲染器都会"注入"功能的实现(如`setState`)进入通用的 React 包，以使您的组件更具表诉性。

使用 React 时，您无需考虑其工作原理。我们希望 React 用户花更多时间考虑他们的应用程序代码，而不是像依赖注入这样的抽象概念。但如果你曾经想过，`this.setState()`或`useState()`是如何知道该怎么做？这个问题，那我希望这篇博文会有所帮助。

---
