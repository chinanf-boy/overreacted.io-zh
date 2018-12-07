---
title: 到底 React 怎么 以一个 function ，来推断为一个 class?
date: '2018-12-02'
spoiler: 我们来翻译翻译 classes, new, instanceof, prototype chains, 和 API 设计.
---

考虑这个定义为函数的`Greeting`组件:

```js
function Greeting() {
  return <p>Hello</p>
}
```

React 还支持将它定义为类:

```js
class Greeting extends React.Component {
  render() {
    return <p>Hello</p>
  }
}
```

(直到[不久前](https://reactjs.org/docs/hooks-intro.html)，这是使用状态(state)的唯一方法.)

当你想要渲染一个`<Greeting />`，你不需要在乎它是如何定义的:

```jsx
// Class 或 function — 随便.
<Greeting />
```

但是*React 自身*会关心其差异!

如果`Greeting`是一个函数,React 需要调用它:

```js
// 你的码
function Greeting() {
  return <p>Hello</p>
}

// React 内部
const result = Greeting(props) // <p>Hello</p>
```

但，如果`Greeting`是一个类,React 需要用`new`运算符实例化它，和*然后*在刚才创建的实例内，调用`render`方法:

```js
// 你的码
class Greeting extends React.Component {
  render() {
    return <p>Hello</p>
  }
}

// React 内部
const instance = new Greeting(props) // Greeting {}
const result = instance.render() // <p>Hello</p>
```

在这两种情况下,React 的目标都是得到渲染的节点(在这个例子中,`<p>Hello</p>`)，但是确切的步骤取决于`Greeting`的定义。

**那么,React 是如何知道某事物是一个类，还是一个函数的呢?**

就像我的[上一篇博文](/why-do-we-write-super-props/)所说，**你不*需要*知道这个，也能用好 React.**这几年我都不知道。请不要把这个变成面试题。事实上,这篇文章更多的是关于 JavaScript,而不是关于 React.

这个博客应该是，一个好奇的读者想知道，_为什么_ React 以某种方式工作？你是那个人吗? 那我们一起挖吧。

**这是一段漫长的旅程。扣上安全带。这篇文章没有关于 React 本身的很多信息,但是我们接下来会介绍几个方面，有关`new`，`this`，`class`，箭头函数，`prototype`，`__proto__`，`instanceof`以及这些事情是如何在 JavaScript 中协同工作的。幸运的是,你根本不需要考虑这些东东，才能*使用*React。但如果你正在实现 React…(当我没说)**

(如果你真的想知道答案,滚动到最后.)

---

首先,我们需要理解为什么区别对待函数和类很重要。注意我们如何使用，调用类时的`new`运算符:

```js{5}
// 若 Greeting 是一个 function
const result = Greeting(props) // <p>Hello</p>

// 若 Greeting 是一个 class
const instance = new Greeting(props) // Greeting {}
const result = instance.render() // <p>Hello</p>
```

让我们粗略地了解一下，`new`运算符在 JavaScript 做了神秘.

---

在过去啊，JavaScript 没有 class。但是,您可以使用简单函数来表示与类相似的模式。**具体地说,你可以使用*任何*类似-类构造函数的函数，然在调用前，添加`new`就好:**

```js
// Just a function
function Person(name) {
  this.name = name
}

var fred = new Person('Fred') // ✅ Person {name: 'Fred'}
var george = Person('George') // 🔴 不工作
```

你今天仍然可以编写这样的代码!在 DevTools(调试器) 试试看。

如果你用`Person('Fred')`，带 **没有** `new`，那`this`里面会指向一些全局的和无用的东西(例如,`window`或`undefined`)，所以我们的代码会崩溃或者做一些像设置`window.name`的愚蠢事情.

另一种是，添加了`new`,我们说:"嘿,JavaScript,我知道`Person`只是一个函数，但我们假设它是一个类构造函数。**创建一个`{}`对象与在`Person`函数里面`this`指向建好的对象，这样我就可以分配像`this.name`的东西。 然后把那个对象还给我.**"

这即是`new`运算符做的.

```js
var fred = new Person('Fred') // `Person`里面的`this`，是一样的对象
```

这个`new`运算符也会让，我们对`Person.prototype`所做的事情成功，应用到`fred`对象:

```js{4-6,9}
function Person(name) {
  this.name = name
}
Person.prototype.sayHi = function() {
  alert('Hi, I am ' + this.name)
}

var fred = new Person('Fred')
fred.sayHi()
```

在 JavaScript 直接添加类之前，人们就是这么模拟类。

---

所以`new`在 JavaScript 中已经有一段时间了。然而,类是最近的。他们让我们重写上面的代码，以更贴近我们的意图:

```js
class Person {
  constructor(name) {
    this.name = name
  }
  sayHi() {
    alert('Hi, I am ' + this.name)
  }
}

let fred = new Person('Fred')
fred.sayHi()
```

_知晓开发者意图_，在语言和 API 设计中很重要.

如果你写了一个函数，JavaScript 不能猜测它是否为`alert()`被调用，或者是用`new Person()`作为构造函数。忘记对`Person`函数指定`new`会导致混乱的行为。

**类(Class)语法让我们可以说:"这不仅仅是一个函数——它是一个类,它有一个构造函数".**如果你调用它时忘记了`new`，JavaScript 会引发错误:

```js
let fred = new Person('Fred')
// ✅  若 Person 是一个 function: 可以工作
// ✅  若 Person 是一个 class: 一样可以工作

let george = Person('George') // 忘了加 `new`
// 😳 若 Person 是一个 类构造(constructor) function: 行为混乱
// 🔴 若 Person 是一个 class: 立即错误
```

这有助于我们尽早捕捉错误,而不是等待一些模糊的 bug，如`this.name`被视为`window.name`，而不是`george.name`。

但是,这意味着 React 需要在执行任何类之前，放置`new`。它不能像常规函数那样调用，因为 JavaScript 会将其视为错误!

```js
class Counter extends React.Component {
  render() {
    return <p>Hello</p>
  }
}

// 🔴 React 不能这么做:
const instance = Counter(props)
```

这意味着麻烦。

---

在我们看到 React 如何解决这个问题之前，有个重点注意的是，大多数人通过使用像 Babel 这样的编译器来使用 React ，因为 Babel 可以帮助我们 编译现代函数，比如用于旧浏览器的类。所以我们需要在设计中考虑编译器.

在 Babel 的早期版本中,可以在没有`new`的情况下调用类。但,这是硬编码的 - 通过生成一些额外的代码:

```js
function Person(name) {
  // Babel 输出(简化版):
  if (!(this instanceof Person)) {
    throw new TypeError('Cannot call a class as a function')
  }
  // Our code:
  this.name = name
}

new Person('Fred') // ✅ Okay
Person('George') // 🔴 Can’t call class as a function
```

您可能在捆绑(js)包中看到过类似的代码。这就是所有这些`_classCallCheck`函数要做的。(您可以通过选择进入"松散模式(loose mode)"，而不进行检查，来减小捆绑包大小，但这可能会使您最终转换为真正的原生类变得复杂.)

---

到现在为止,你应该大致了解有`new`或没有`new`，调用某某之间的区别:

|            | `new Person()`           | `Person()`                        |
| ---------- | ------------------------ | --------------------------------- |
| `class`    | ✅`this`是一个`Person`例 | 🔴`TypeError`                     |
| `function` | ✅`this`是一个`Person`例 | 😳`this`是`window`要么`undefined` |

这就是为什么 React 要正确调用组件的重要原因。**如果您的组件被定义为类,那 React 就需要使用`new`调用.**

那么 React 可以检查一下是不是某个类?

不是那么容易的! 即使我们可以[在 JavaScript 中，以一个 function 来推断为一个 class](https://stackoverflow.com/questions/29093396/how-do-you-check-the-difference-between-an-ecmascript-6-class-and-function)，但仍然不适用于像 Babel 这样的工具处理过的类。对于浏览器来说，它们只是普通的函数。祝你好运啦，React 。

---

好吧，那我让, React 在每次调用时，无脑使用`new`? 不幸的是,这并不总是奏效.

使用常规函数,用`new`调用它们会给他们一个`this`的对象实例。这种方式，给本就编写为'构造函数'的函数是明智的(就像我们上面的`Person`),但它会混淆函数组件:

```js
function Greeting() {
  // 我们不要有任何实例类型的`this`，在这里
  return <p>Hello</p>
}
```

但，这又好像是可以容忍的。那我只能告诉，下面有两个掐掉这个想法的*其他*原因.

---

第一个原因: 始终使用`new`对原生箭头函数是不会起作用(不是由 Babel 编译的函数)，其`new`调用时，就会抛出错误:

```js
const Greeting = () => <p>Hello</p>
new Greeting() // 🔴 Greeting 不是一个构造函数
```

这种行为是有原因的,并且遵循箭头函数的设计。箭头函数的主要优点之一就是它们*不具有*自己的`this`值 - 相反,`this`从最接近的常规函数 ​​ 获取:

```js{2,6,7}
class Friends extends React.Component {
  render() {
    const friends = this.props.friends
    return friends.map(friend => (
      <Friend
        // `this` 来自 `render` 方法
        size={this.props.size}
        name={friend.name}
        key={friend.id}
      />
    ))
  }
}
```

嗯，所以**箭头函数没有自己的`this`.**，但也意味着他们作为构造函数将完全无用!

```js
const Person = name => {
  // 🔴 一个浮动的 this!，又如何明确知道.name
  this.name = name
}
```

因此,**JavaScript 不允许`new`调用箭头函数。** 如果你这样做,无论如何你都可能犯了一个错误，且最早告诉你。这与 JavaScript 不允许您不带`new`调用类的方式类似.

这思路很不错，但它也影响了我们的计划。React 不能只是`new`一切，因为它会打破箭头函数! 另作打算，我们可以尝试通过它们缺乏的`prototype`来检测箭头函数, 并不仅是`new`他们:

```js
;(() => {}).prototype(
  // undefined
  function() {}
).prototype // {constructor: f}
```

但是这对使用 Babel 编译的函数[不能够成功](https://github.com/facebook/react/issues/4599#issuecomment-136562930)。你想这可能不是什么大问题，但还有另一个原因让这种方法成为死胡同.

---

我们不能总是使用`new`的另一个原因是，它会阻止 React 支持，可返回字符串或其他原始类型的组件.

```js
function Greeting() {
  return 'Hello'
}

Greeting() // ✅ 'Hello'
new Greeting() // 😳 Greeting {}
```

这，再一次，与[`new` 运算符](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new)设计这个问题有关。正如我们之前看到的,`new`告诉 JavaScript 引擎创建一个对象，使该对象`this`在函数内部，后来给会我们那个对象，作为`new`的一个结果。

但是，JavaScript 也允许`new`调用函数，会*覆盖*`new`返回值，通过返回一些其他对象。你想啊，这样的模式对于我们想要重用的实例'池'会很有用:

```js{1-2,7-8,17-18}
// Created lazily
var zeroVector = null

function Vector(x, y) {
  if (x === 0 && y === 0) {
    if (zeroVector !== null) {
      // 重用同一个 实例
      return zeroVector
    }
    zeroVector = this
  }
  this.x = x
  this.y = y
}

var a = new Vector(1, 1)
var b = new Vector(0, 0)
var c = new Vector(0, 0) // 😲 b === c
```

然而，如果返回的*不*是一个对象，`new`也会*完全无视了*一个函数的返回值。如果你返回一个字符串或数字，就像完全没有`return`一样。

```js
function Answer() {
  return 42
}

Answer() // ✅ 42
new Answer() // 😳 Answer {}
```

在`new`调用函数时,无法从函数中读取原始返回值(如数字或字符串)。所以如果 React 总是使用`new`的话，它将无法添加，返回字符串的支持组件!

这是不可接受的，所以我们需要妥协。

---

到目前为止我们学到了什么?React 需要`new`调用类(包括 Babel 输出)，但它要让常规函数或箭头函数(包括 Babel 输出)_不带_`new`调用。且是没有可靠的方法来区分它们.

**如果我们无法解决一个通用问题，我们能解决一个更具体的问题吗?**

当你将组件定义为类时，您可能希望扩展`React.Component`内置方法，如`this.setState()`。**我们可以只检测`React.Component`后辈,而不是尝试检测所有类吗?**

剧透:这正是 React 所做的.

---

也许,惯用的检查方法是，若`Greeting`是一个 React 组件类，可通过测试`Greeting.prototype instanceof React.Component`结果:

```js
class A {}
class B extends A {}

console.log(B.prototype instanceof A) // true
```

我知道你在想什么。**刚**刚发生了什么?! 要回答这个问题,我们需要了解 JavaScript 原型.

您可能熟悉"原型链"。JavaScript 中的每个对象都可能有一个"原型"。我们在写`fred.sayHi()`的时候，可是`fred`对象并没有`sayHi`属性啊，我们寻找`fred`的原型的`sayHi`属性。如果我们在那里找不到它,我们会看看链中的下一个原型 -`fred`原型的原型。等等等。

**令人困惑的是，一个类或函数的`prototype`属性*才不是*指向该值的原型(`prototype`).** 我不是在开玩笑(往下看)。

```js
function Person() {}

console.log(Person.prototype) // 🤪 不是 Person的 prototype
console.log(Person.__proto__) // 😳 Person的 prototype
```

所以"原型链"更像是`__proto__.__proto__.__proto__`，不是`prototype.prototype.prototype`。这花了我多年才知道。

那到底`prototype`属性是函数或类的什么呢? **正确来说`prototype`属性是，给予`__proto__`所有`new`了的类或函数的对象！**

```js{8}
function Person(name) {
  this.name = name
}
Person.prototype.sayHi = function() {
  alert('Hi, I am ' + this.name)
}

var fred = new Person('Fred') // 将 `fred.__proto__` 设为了 `Person.prototype`
```

然后`__proto__`链，是 JavaScript 查找属性的方式:

```js
fred.sayHi()
// 1.  fred 有一个 sayHi 属性吗? 没有.
// 2.  fred.__proto__ 有一个 sayHi 属性吗? 有啊. Call it!

fred.toString()
// 1. fred 有一个 toString 属性吗? No.
// 2. fred.__proto__ 有一个 toString 属性吗? No.
// 3. fred.__proto__.__proto__ 有一个 toString 属性吗? Yes. Call it!
```

在代码实践中，你几乎不需要直接触摸`__proto__`，除非您正在调试与原型链相关的内容。如果你想为`fred.__proto__`直接，你应该在`Person.prototype`这里，帮它穿上'衣服'。至少这是它最初设计的方式。

因为原型链被认为是一个内部概念，因此该`__proto__`甚至不应该被浏览器暴露出来。但有些浏览器补充了`__proto__`，最终它被勉强标准化(但，使用`Object.getPrototypeOf()`是举双手赞成的)。

**但是啊，我仍然觉得一个叫做`prototype`属性的东东，它却没有给你一个值的原型(`prototype`)，相当让人困惑**(例如,`fred.prototype`是未定义(undefined)，因为`fred`不是一个函数)。就个人而言，我认为这是即便经验丰富的开发人员，也会误解 JavaScript 原型的最大原因。

---

这是一个很长的帖子,嗯? 但我只能说，我们现在写了 80%。支持住啊。

我们知道,当请求`obj.foo`是，JavaScript 实际寻找的`foo`的路径顺序是,`obj`,`obj.__proto__`,`obj.__proto__.__proto__`, 等等.

而对于类，您不会直接暴露于此机制，但是优秀的`extends`也适用于旧原型链。这就是我们 React 类实例，怎样访问类似`setState`方法的方式:

```js{1,9,13}
class Greeting extends React.Component {
  render() {
    return <p>Hello</p>
  }
}

let c = new Greeting()
console.log(c.__proto__) // Greeting.prototype
console.log(c.__proto__.__proto__) // React.Component.prototype
console.log(c.__proto__.__proto__.__proto__) // Object.prototype

c.render() // Found on c.__proto__ (Greeting.prototype)
c.setState() // Found on c.__proto__.__proto__ (React.Component.prototype)
c.toString() // Found on c.__proto__.__proto__.__proto__ (Object.prototype)
```

换一种说法,**当你使用类时,一个实例的`__proto__`链"镜像"了类的层次结构:**

```js
// `extends` 链
Greeting
  → React.Component
    → Object (implicitly)

// `__proto__` 链
new Greeting()
  → Greeting.prototype
    → React.Component.prototype
      → Object.prototype
```

双链-关联。

<!-- HERE -->

---

自从`__proto__`链，镜像了类的层次结构，我们可以检查一下`Greeting`来自`React.Component`的扩展，顺序从`Greeting.prototype`开始，然后跟着它`__proto__`链往下:

```js{3,4}
// `__proto__` 链
new Greeting()
  → Greeting.prototype // 🕵️ 我们从这里开始
    → React.Component.prototype // ✅ Found it!
      → Object.prototype
```

方便的方法是,`x instanceof Y`，其正是这种搜索。它遵循`x.__proto__`链，在那里寻找`Y.prototype`。

通常用于确定某些东西是否，为类的实例:

```js
let greeting = new Greeting()

console.log(greeting instanceof Greeting) // true
// greeting (🕵️‍ We start here)
//   .__proto__ → Greeting.prototype (✅ Found it!)
//     .__proto__ → React.Component.prototype
//       .__proto__ → Object.prototype

console.log(greeting instanceof React.Component) // true
// greeting (🕵️‍ We start here)
//   .__proto__ → Greeting.prototype
//     .__proto__ → React.Component.prototype (✅ Found it!)
//       .__proto__ → Object.prototype

console.log(greeting instanceof Object) // true
// greeting (🕵️‍ We start here)
//   .__proto__ → Greeting.prototype
//     .__proto__ → React.Component.prototype
//       .__proto__ → Object.prototype (✅ Found it!)

console.log(greeting instanceof Banana) // false
// greeting (🕵️‍ We start here)
//   .__proto__ → Greeting.prototype
//     .__proto__ → React.Component.prototype
//       .__proto__ → Object.prototype (🙅‍ Did not find it!)
```

它也可以，确定一个类是否扩展了另一个类:

```js
console.log(Greeting.prototype instanceof React.Component)
// greeting
//   .__proto__ → Greeting.prototype (🕵️‍ We start here)
//     .__proto__ → React.Component.prototype (✅ Found it!)
//       .__proto__ → Object.prototype
```

这种方式，我们也能用来，确认某些东西是否 React 组件类或常规函数。

---

但这并不是 React 所做的.😳

`instanceof`解决方案的一个警告是，当页面上有多个 React 副本，并且我们正在检查的组件，来自*另一个*React 复制的`React.Component`继承时，它不起作用。在一个项目中混用多个 React 副本是不好的，原因有几个,但从历史上看,我们尽可能避免出现问题。(有了 Hooks,我们[也需要需要](https://github.com/facebook/react/issues/13991)对重复数据强制删除。)

另一种可能的启发式方法可能是检查原型上，是否存在`render`方法。但是, 到了怎样发展组件 API 的那个时候呢，又会[不清不楚](https://github.com/facebook/react/issues/4599#issuecomment-129714112)。每种检查方式(支票)都有其成本，所以我们不想添加多张。如果`render`被定义为实例方法，这也行不通，原因有像，使用了类属性语法。

相反,React[增加了](https://github.com/facebook/react/pull/4663)一个基本组件的特殊标志。React 通过检查是否存在该标志，以此，它知道某些东西是否为 React 组件类.

最初的标志，在基本`React.Component`类里面:

```js
// React 内部
class Component {}
Component.isReactClass = {}

// 我们可以像这样，检查它
class Greeting extends Component {}
console.log(Greeting.isReactClass) // ✅ Yes
```

但是,我们有一些类的实现定位，使其[不能够](https://github.com/scala-js/scala-js/issues/1900)复制静态属性(或设置非标准`__proto__`)，所以该标志会迷路。

这就是 React 的原因[移动](https://github.com/facebook/react/pull/5021)这个标志到 `React.Component.prototype`:

```js
// React 内部
class Component {}
Component.prototype.isReactComponent = {}

// 我们这样，做检查
class Greeting extends Component {}
console.log(Greeting.prototype.isReactComponent) // ✅ Yes
```

**而这就是字面意思(是 React 组件-isReactComponent).**

您可能想知道为什么它是一个对象,而不仅仅是一个布尔值。在实践中，这并不重要,但是早期版本的 Jest(在之前 Jest 是 Good™️)默认启用了自动模拟。生成的模拟省略了原始属性，[破环了该检查](https://github.com/facebook/react/pull/4663#issuecomment-136533373)。 谢谢啊！！！, Jest.

这个`isReactComponent`检查，已是[React](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L297-L300)使用至今。

如果你不(extend)扩展`React.Component`，React 在原型上是找不到`isReactComponent`，也就不会把组件当作一个类来对待。现在你知道为什么，`Cannot call a class as a function`问题的[置顶的答案](https://stackoverflow.com/a/42680526/458193)，就是添加`extends React.Component`的原因了吧。 最后,一个[警告被增加了](https://github.com/facebook/react/pull/11168)，这个警告是说，当`prototype.render`存在，但`prototype.isReactComponent`却不存在. 这么个意思。

---

你可能会说这个故事有点诱饵和开关。**实际的解决方法很简单嘛，但我‘高度’地解释了*为什么?*React 最后使用了这个解决方案,以及替代方案是什么.**

根据我的经验,框架 API 通常是这样争论的。要使 API 易于使用，您通常需要考虑语言语义(可能对于几种语言,包括未来方向)、运行时性能、编译步骤具不具有人机工程学、生态系统状态和包装解决方案、早期警告,以及许多其他的。所以啊，最终结果可能并不总是最优雅的,但它必须是实用的。

**如果最终 API 是成功，*其用户*永远都不必考虑这个过程.**相反,他们可能更专注于创建好应用程序。

但是，如果你同样好奇… 知道了它是如何工作，还是会很高兴的 :)
