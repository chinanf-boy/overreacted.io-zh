---
title: 为什么React Hooks 依赖于 调用顺序?
date: '2018-12-13'
spoiler: 从 mixins, render props, HOCs, 和 classes 学到的经验教训.
---

在 React Conf 2018 上,React 团队展示了[Hooks 提案](https://reactjs.org/docs/hooks-intro.html).

如果你想了解 Hooks 是什么，它们解决了什么问题，请查看我们[介绍性的讲座](https://www.youtube.com/watch?v=dpw9EHDh2bM)和我解释常见的误解的[后续博文](https://medium.com/@dan_abramov/making-sense-of-react-hooks-fdbde8803889).

一开始，你可能不喜欢 Hooks(下图为，个人的否定言论):

![Negative HN comment](./hooks-hn1.png)

它们就像一张音乐唱片，只有在几次静静地聆听后，才会让你感觉成长(下图是，同个人的'真香'-肯定言论):

![Positive HN comment from the same person four days later](./hooks-hn2.png)

当你读此文的时候，不要错过[最重要的文档](https://reactjs.org/docs/hooks-custom.html)，其有建立你自己的 Hooks 的相关信息! 大多数人过分把目光放在，表示他们不同意的部分信息上(例如，学习课程很难)，而这，会错过 Hooks 背后更大的版图，就是**Hooks 会像*功能性融合(mixins)*，能让你可以创建和撰写你自己的抽象。**

Hooks[由一些现有技术所引导的](https://reactjs.org/docs/hooks-faq.html#what-is-the-prior-art-for-hooks)，但我没见过像他们**这样的**，直到 Sebastian 与团队分享了他的想法。不幸的是，这个设计很容易忽略，特定 API 选择，与不锁定的值属性之间的连接。通过这篇文章，我希望能帮助更多的人理解 Hooks 提案中，最具争议部分的理由。

**本文的其余部分假设您知道`useState()`Hooks API 以及如何编写自定义 Hooks。如果不清楚，请查看前面的链接。另外,记住 Hooks 是实验性的，你现在不必学习它们!**

(免责声明:这是个人帖子，不一定反映 React 团队的意见。它很大，是个复杂主题，我可能会在某个地方犯了错误。)

---

当你了解 Hooks 的时候，第一个和可能也是最大的惊讶，是它们居然依赖*重渲染器之间持续调用的索引*。 这有一些[实现](https://reactjs.org/docs/hooks-rules.html)。

这个决定显然有争议。这就是为什么，[根据我们的原则](https://www.reddit.com/r/reactjs/comments/9xs2r6/sebmarkbages_response_to_hooks_rfc_feedback/e9wh4um/)，我们只有在感觉文档和会谈，对它描述的足够好之后，想着让人们给它一个公平的机会，才发表了这个提案。

**如果您关心 Hooks API 设计的某些方面，我建议您阅读 Sebastian 的[完整 回复](https://github.com/reactjs/rfcs/pull/68#issuecomment-439314884)给 1000 多条评论的 RFC 讨论。**它很通透，但也相当密集。我可能会把这篇评论的每一段，都变成自己的博客文章。(事实上，我已经[做](/how-does-setstate-know-what-to-do/)了一次!)

今天我有一个特别的部分要关注。您可能还记得，每个 Hooks 可以在一个组件中，使用多次。例如，我们可以声明[多个 state 变量](https://reactjs.org/docs/hooks-state.html#tip-using-multiple-state-variables)，通过重复调用`useState()`:

```jsx{2,3,4}
function Form() {
  const [name, setName] = useState('Mary') // State variable 1
  const [surname, setSurname] = useState('Poppins') // State variable 2
  const [width, setWidth] = useState(window.innerWidth) // State variable 3

  useEffect(() => {
    const handleResize = () => setWidth(window.innerWidth)
    window.addEventListener('resize', handleResize)
    return () => window.removeEventListener('resize', handleResize)
  })

  function handleNameChange(e) {
    setName(e.target.value)
  }

  function handleSurnameChange(e) {
    setSurname(e.target.value)
  }

  return (
    <>
      <input value={name} onChange={handleNameChange} />
      <input value={surname} onChange={handleSurnameChange} />
      <p>
        Hello, {name} {surname}
      </p>
      <p>Window width: {width}</p>
    </>
  )
}
```

注意,我们使用数组分构语法，来命名`useState()`状态变量，但这些名称不会传递给 react。相反，在这个例子中**React 将`name`处理成"第一状态变量"，`surname`处理成"第二状态变量"，等等**。他们的*调用索引*是它们在重新渲染之间的稳定标示。这个思维模型[在这个文章中](https://medium.com/@ryardley/react-hooks-not-magic-just-arrays-cd4f1857236e)描述得很好。

表面来看，依靠着 调用索引 就是*感觉不对*。 直觉是一个有用的信号，但它也有可能误导我们——特别是，如果我们没有完全消化我们正在解决的问题。**在这篇文章中，我将采取一些常见的设计建议替代 Hooks，并展示他们在哪里发生故障。**

---

这篇文章并不详尽。根据你计算的精确程度，我们已经看到了 十几种 到*数百*种不同的备选方案。在过去五年里，我们也一直在[考虑](https://github.com/reactjs/react-future)替代组件 API.

如此类的博客文章会很棘手，因为即使你涵盖了 100 个备选方案，也有人可以调整一个，然后说:"哈,你没想到*这*啊!"

在实践中，不同的替代方案往往在缺点上重叠。我会用典型的例子示范最常见的缺陷，而不是列举*所有的*建议 API(需要几个月的时间)。通过这些问题对其他可能的 API 进行分类，对读者来说可能是一个练习。

*这并不是说 Hooks 是完美的。*但是一旦你熟悉了其他解决方案的缺陷，你可能会发现 Hooks 的设计是有一定道理的。

---

### 缺陷 #1: 无法提取自定义 Hook

<!-- HERE -->

令人惊讶的是，许多备选方案完全都不允许[自定义 Hooks](https://reactjs.org/docs/hooks-custom.html)。也许我们在"动机"一文中，没有足够强调自定义 Hooks 的重要性。因为在很好地理解 Hooks 提案 之前，这很难做到。所以这是一个先有鸡，还是先有蛋的问题。但是自定义 Hooks 在很大程度上是这个提案的要点。

例如，在组件中，备用方案会禁止多个`useState()`调用。你可以在一个对象中保存状态(state)。这同样适用于 class，对吧?

```jsx
function Form() {
  const [state, setState] = useState({
    name: 'Mary',
    surname: 'Poppins',
    width: window.innerWidth,
  })
  // ...
}
```

要清楚的是，Hooks*确实*允许此样式。但你不一定*必须*将您的状态拆分为一组状态变量(请参见, FAQ 中的[建议](https://reactjs.org/docs/hooks-faq.html#should-i-use-one-or-many-state-variables)).

但，支持多个`useState()`调用是为了，让你能*提取*部分的状态性逻辑(状态+效果)，让其从组件中释放到自定义 Hooks ，该 Hooks *也*可以独立使用当地状态和效果:

```jsx{6-7}
function Form() {
  // Declare some state variables directly in component body
  const [name, setName] = useState('Mary')
  const [surname, setSurname] = useState('Poppins')

  // We moved some state and effects into a custom Hook
  const width = useWindowWidth()
  // ...
}

function useWindowWidth() {
  // Declare some state and effects in a custom Hook
  const [width, setWidth] = useState(window.innerWidth)
  useEffect(() => {
    // ...
  })
  return width
}
```

如果你只允许每个组件调用一个`useState()`，您将失去自定义 Hooks 引入本地状态的能力。这是自定义 Hooks 的关键点。

### 缺陷 #2: 名称 冲突

一个常见的建议是，让`useState()`在组件中接受键名称(key)参数(例如字符串)，这样就可以唯一标识一个指定的状态变量。

这里有几个不同点，但大致如下:

```jsx
// ⚠️ 这 不是  React Hooks API
function Form() {
  // 我们 传递 一些 类型 state key 给 useState()
  // 👌 key 就是那些 ('name') , ('surname') , ('width')
  const [name, setName] = useState('name');
  const [surname, setSurname] = useState('surname');
  const [width, setWidth] = useState('width');
  // ...
```

这试图避免依赖调用索引(多 明显的 keys!)，但也带来了另一个问题——名称冲突.

<!-- HERE -->

当然，除不小心错了之外，你可能不会想在同一组件中调用`useState('name')`两次。这可能是偶而发生的，但我们可以与任何错误抗争。然而，这与你在*自定义 Hooks*上工作时差不多，您会想要添加或删除状态变量和效果。

有了这个建议，无论你何时在自定义 Hooks 中添加一个新的状态变量，都有可能破坏任何使用它的组件(直接或传递)，因为*他们可能已经用了相同的名字*表明他们自己的状态变量。

这不是一个[针对更改，进行优化](/optimized-for-change/)的 API 示例。 当前的代码可能看起来总是"优雅"，但是它对于需求的变化是非常脆弱的。我们应该从我们的错误中[学取教训](https://reactjs.org/blog/2016/07/13/mixins-considered-harmful.html#mixins-cause-name-clashes)。

实际上， Hooks 提案通过依赖调用顺序来解决这个问题: 即使两个 Hooks 都使用`name`作为状态变量，它们也会相互独立。每个`useState()`调用有自己的"内存单元"。

我们还有其他一些方法可以解决这个缺陷，但它们也有自己的问题。让我们更深入探索这个问题区。

### 缺陷 #3: 不能调用同个 Hook 两次

另一个"keyed(已用)"`useState`提案的变体，是建议使用类似 Symbols 的东西。这样就不会冲突了，对吧?

```jsx
// ⚠️ 这 不是 React Hooks API
const nameKey = Symbol();
const surnameKey = Symbol();
const widthKey = Symbol();

function Form() {
  // 我们 传递 一些 类型 state key 给 useState()
  const [name, setName] = useState(nameKey);
  const [surname, setSurname] = useState(surnameKey);
  const [width, setWidth] = useState(widthKey);
  // ...
```

这个建议似乎对提取`useWindowWidth()`Hook 起作用:

```jsx{4,11-17}
// ⚠️ 这 不是 React Hooks API
function Form() {
  // ...
  const width = useWindowWidth()
  // ...
}

/*********************
 * useWindowWidth.js *
 ********************/
const widthKey = Symbol()

function useWindowWidth() {
  const [width, setWidth] = useState(widthKey)
  // ...
  return width
}
```

但如果我们尝试提取输入处理，它将失败:

```jsx{4,5,19-29}
// ⚠️ 这 不是 React Hooks API
function Form() {
  // ...
  const name = useFormInput()
  const surname = useFormInput()
  // ...
  return (
    <>
      <input {...name} />
      <input {...surname} />
      {/* ... */}
    </>
  )
}

/*******************
 * useFormInput.js *
 ******************/
const valueKey = Symbol()

function useFormInput() {
  const [value, setValue] = useState(valueKey)
  return {
    value,
    onChange(e) {
      setValue(e.target.value)
    },
  }
}
```

(我承认这一点`useFormInput()`Hooks 不是特别有用，但您可以想象它处理诸如验证和不规状态标志一个[Formik](https://github.com/jaredpalmer/formik)之类的事情.)

你能发现 bug 吗?

我们在调用`useFormInput()`两次，但`useFormInput()`总是执行`useState()`用同一 key。所以实际上，我们是做了如下一些事情:

```jsx
const [name, setName] = useState(valueKey)
const [surname, setSurname] = useState(valueKey)
```

这就是我们再次发生冲突的原因。

实际的 Hooks 提案没有这个问题，因为**每个到`useState()`的 _调用_ 都获取自己的独立状态。**依靠一个持久调用索引可以使我们免于担心名称冲突。

### 缺陷 #4: Diamond(钻石) 问题

这在技术上与前一个相同，但值得一提的是它的臭名昭着。他甚至[在维基百科有所描述](https://en.wikipedia.org/wiki/Multiple_inheritance#The_diamond_problem)。
(很显然，它有时被称为“死亡的致命钻石 - cool beans)

我们自己的 mixin 系统[就遭遇到了它](https://reactjs.org/blog/2016/07/13/mixins-considered-harmful.html#mixins-cause-name-clashes).

在引擎盖下面，两个自定义 Hooks 像`useWindowWidth()`和`useNetworkStatus()`，可能想要使用相同的自定义 Hook`useSubscription()`:

```jsx{12,23-27,32-42}
function StatusMessage() {
  const width = useWindowWidth()
  const isOnline = useNetworkStatus()
  return (
    <>
      <p>Window width is {width}</p>
      <p>You are {isOnline ? 'online' : 'offline'}</p>
    </>
  )
}

function useSubscription(subscribe, unsubscribe, getValue) {
  const [state, setState] = useState(getValue())
  useEffect(() => {
    const handleChange = () => setState(getValue())
    subscribe(handleChange)
    return () => unsubscribe(handleChange)
  })
  return state
}

function useWindowWidth() {
  const width = useSubscription(
    handler => window.addEventListener('resize', handler),
    handler => window.removeEventListener('resize', handler),
    () => window.innerWidth
  )
  return width
}

function useNetworkStatus() {
  const isOnline = useSubscription(
    handler => {
      window.addEventListener('online', handler)
      window.addEventListener('offline', handler)
    },
    handler => {
      window.removeEventListener('online', handler)
      window.removeEventListener('offline', handler)
    },
    () => navigator.onLine
  )
  return isOnline
}
```

这是一个完全有效的用例。**对于自定义 Hook 作者来说，启动或停止使用另一个自定义 Hook 应该是安全的，而不必担心它是否已在链中某处“已经使用”** 实际上，除非您在每次更改时使用 Hook 审核每个组件，否则您*永远无法了解*整个链。

(作为一个反例，React 遗留的 `createClass()`mixin 没有让你这样做。有时你会有两个 mixin，它们都完全符合你的需要，但由于扩展了相同的“base”mixin ,而互不兼容。)

这是我们的“钻石”：💎

```
       / useWindowWidth()   \                   / useState()  🔴 冲突
Status                        useSubscription()
       \ useNetworkStatus() /                   \ useEffect() 🔴 冲突
```

依赖于持续的调用顺序，自然会解决它:

```
                                                 / useState()  ✅ #1. State
       / useWindowWidth()   -> useSubscription()
      /                                          \ useEffect() ✅ #2. Effect
Status
      \                                          / useState()  ✅ #3. State
       \ useNetworkStatus() -> useSubscription()
                                                 \ useEffect() ✅ #4. Effect
```

函数调用没有“钻石”问题，因为它们形成了一个树。🎄

### 缺陷 #5: 复制粘贴破坏事物

也许我们可以通过引入某种命名空间来挽救 keyed state 提案。有几种不同的方法可以做到这一点。

一种方法是使用闭包隔离状态键(key)。这将要求您“实例化”自定义 Hook，并在每个 Hook 周围添加一个函数包装器：

```jsx{5,6}
/*******************
 * useFormInput.js *
 ******************/
function createUseFormInput() {
  // Unique per instantiation
  const valueKey = Symbol()

  return function useFormInput() {
    const [value, setValue] = useState(valueKey)
    return {
      value,
      onChange(e) {
        setValue(e.target.value)
      },
    }
  }
}
```

这种做法下非常重手。Hooks 的设计目标之一是，避免使用高阶组件和渲染 props 所普遍存在的深层嵌套函数样式。在这里，我们不得不在使用前，“实例化” 的任何自定义 Hook - 并在一个组件内使用所产生的函数*正好一次*。这并不比无条件地调用 Hook 简单多少。

此外，您必须重复两次使用组件中的每个自定义 Hook。一次在顶层范围（或者在函数范围内，若是我们编写自定义 Hook 了的话），和一次在实际调用点。这意味着即使是小的更改，您也必须在渲染和顶层声明之间跳转：

```js{2,3,7,8}
// ⚠️ 这 不是 React Hooks API
const useNameFormInput = createUseFormInput()
const useSurnameFormInput = createUseFormInput()

function Form() {
  // ...
  const name = useNameFormInput()
  const surname = useNameFormInput()
  // ...
}
```

你还需要非常精确地说出他们的名字。你会一直有“两个层级”的名称 - 像`createUseFormInput`工厂，还有实例化 Hooks 的`useNameFormInput`与`useSurnameFormInput`。

如果您调用相同的自定义 Hooks"实例"两次，冲突会发生。事实上，上面的代码有个错误 - 不知道你有没注意到? 其实正确的是:

```js
const name = useNameFormInput()
const surname = useSurnameFormInput() // Not useNameFormInput!
```

这些问题并不是不可克服的，但，我认为它们会比遵循[Hooks 规则](https://reactjs.org/docs/hooks-rules.html)，更具摩擦。

重要的是，它们打破了复制粘贴的期望。在没有额外的封装包装的情况下，提取一个自定义 Hook 的方法仍然可以使用，但只在您调用它两次之前可用。（这就是它产生冲突的时候。）不幸的是，当一个 API 看起来有效，可若你意识到在链的某个地方存在冲突时，就会强迫你把所有的东西 ™️ 包裹起来。

### 缺陷 #6: 我们仍需要 一个 Linter

<!-- HERE -->

还有另一种方法可以避免与 keyed state 发生冲突。如果你知道它，你可能会很生气，因为我仍然没有承认它！抱歉。

这个方法是，我们可以在我们每次编写一个自定义 Hook 时*组合* keys。像这样的东西：

```js{4,5,16,17}
// ⚠️ 这 不是 React Hooks API
function Form() {
  // ...
  const name = useFormInput('name')
  const surname = useFormInput('surname')
  // ...
  return (
    <>
      <input {...name} />
      <input {...surname} />
      {/* ... */}
    </>
  )
}

function useFormInput(formInputKey) {
  const [value, setValue] = useState('useFormInput(' + formInputKey + ').value')
  return {
    value,
    onChange(e) {
      setValue(e.target.value)
    },
  }
}
```

出于不同的考虑，我最不喜欢这种方法。我不认为这是值得的。

代码传递非唯一或糟糕的已组合 keys _恰巧能工作_，只因为还没有遇到多次调用 Hook 或与另一个 Hook 发生冲突。更糟糕的是，如果它是有条件的（我们试图“修复”无条件的调用请求，对吧？），我们甚至可能在以后碰到冲突。

记住想在自定义 Hooks 的所有层中传递 key 是很脆弱，为此我们想要提供 lint。他们会帮运行时添加额外的工作（不要忘记他们需要*作为 key*），且每个 lint 都是捆绑包大小的‘剪纸’。**但是，如果我们必须都上 lint，我们又解决了什么问题？**

如果有条件地声明状态和效果是非常可取的，这可能是有意义的。但在实践中我发现它令人困惑。事实上，我不记得有人要求，要有条件地定义`this.state`或者`componentDidMount`其中任何一个。

这段代码是什么意思?

```js{3,4}
// ⚠️ 这 不是 React Hooks API
function Counter(props) {
  if (props.isActive) {
    const [count, setCount] = useState('count');
    return (
      <p onClick={() => setCount(count + 1)}>
        {count}
      </p>;
    );
  }
  return null;
}
```

是为了`props.isActive`等于`false`时，`count`已存在吗? 或是它被重置，因为`useState('count')`没有被调用？

如果 条件下的状态(State) 被保存，效果(Effect)又怎么样？

```js{5-8}
// ⚠️ 这 不是 React Hooks API
function Counter(props) {
  if (props.isActive) {
    const [count, setCount] = useState('count');
    useEffect(() => {
      const id = setInterval(() => setCount(c => c + 1), 1000);
      return () => clearInterval(id);
    }, []);
    return (
      <p onClick={() => setCount(count + 1)}>
        {count}
      </p>;
    );
  }
  return null;
}
```

它在`props.isActive`第一次为`true` _之前_ 绝对不会运行。但，一旦变成`true`，是否意味着它不会停下来？`props.isActive`翻到`false`，又是否重置? **1.** 如果会重置，那令人困惑的是，效果与状态的行为（我们说不会重置）不同。**2.** 如果效果继续运行，那么 `if` 不作用在效果，那就是说不会使效果成为有条件的，这又是很困惑啦。我们不是说，我们想要条件效果吗？

如果在渲染期间，我们没有“使用”状态时，它确实被重置了，那，如果多个包含 `useState('count')` 的 `if` 分支，在任何给定时间只有一个运行，又会发生什么？这真的是有效的代码吗？如果我们的思维模型是“带 keys 的 map”，为什么事情会从中消失？开发人员之后是否期望 组件中的一个早`return`(返回)，能重置所有状态？如果我们真的想要重置状态，我们可以通过提取组件，明确表示：

```jsx
function Counter(props) {
  if (props.isActive) {
    // 明确，自己的状态
    return <TickingCounter />
  }
  return null
}
```

无论如何，这可能就是避免这些令人困惑的问题的“最佳实践”。因此，无论你选择哪种方式来回答这些问题，我认为有条件地*声明*状态和效果本身的语义，最终会变得奇怪，以至于你可能想要 lint 对付它。

如果我们无论如何都需要 lint，正确组成 keys 的要求就变成了“最后一根稻草”。它并没有给我们带来任何我们想要做的事情。但是，放弃这个要求（并回到最初的提案）确能给我们带来了一些东西。它使复制粘贴组件代码到一个自定义 Hook 安全了，无需命名空间，减少了捆绑包大小的剪纸，并解锁了更高效的实现（不需要 Map 查找）。

团结就是力量。

### 缺陷 #7: 无法在 Hooks 之间传递值

Hooks 的一个最好的特性是，可以在它们之间传递值.

下面是一个消息收件人选取器的假设示例，该示例显示当前选择的朋友是否在线:

```jsx{8,9}
const friendList = [
  { id: 1, name: 'Phoebe' },
  { id: 2, name: 'Rachel' },
  { id: 3, name: 'Ross' },
]

function ChatRecipientPicker() {
  const [recipientID, setRecipientID] = useState(1)
  const isRecipientOnline = useFriendStatus(recipientID)

  return (
    <>
      <Circle color={isRecipientOnline ? 'green' : 'red'} />
      <select
        value={recipientID}
        onChange={e => setRecipientID(Number(e.target.value))}
      >
        {friendList.map(friend => (
          <option key={friend.id} value={friend.id}>
            {friend.name}
          </option>
        ))}
      </select>
    </>
  )
}

function useFriendStatus(friendID) {
  const [isOnline, setIsOnline] = useState(null)
  const handleStatusChange = status => setIsOnline(status.isOnline)
  useEffect(() => {
    ChatAPI.subscribeToFriendStatus(friendID, handleStatusChange)
    return () => {
      ChatAPI.unsubscribeFromFriendStatus(friendID, handleStatusChange)
    }
  })
  return isOnline
}
```

当您更改收件人时，我们的`useFriendStatus()`Hook 将取消订阅前一个朋友的状态，并订阅下一个。

可以工作的原因，是我们可以传递`useState()`Hooks 的返回值 到`useFriendStatus()`Hook:

```js{2}
const [recipientID, setRecipientID] = useState(1)
const isRecipientOnline = useFriendStatus(recipientID)
```

在 Hooks 之间传递值的功能非常强大。例如,[React Spring](https://medium.com/@drcmda/hooks-in-react-spring-a-tutorial-c6c436ad7ee4)允许您创建多个值相互"跟随"的尾随动画:

```js
const [{ pos1 }, set] = useSpring({ pos1: [0, 0], config: fast })
const [{ pos2 }] = useSpring({ pos2: pos1, config: slow })
const [{ pos3 }] = useSpring({ pos3: pos2, config: slow })
```

(这里有一个[演示](https://codesandbox.io/s/ppxnl191zx))

将 初始化的 Hook 放入默认参数值或以 装饰器 形式编写 Hook 的建议，会让这种逻辑很难表达。

如果函数体中没有调用 Hooks，就不再能在它们之间轻松地传递值，在不创建多个组件层的情况下转换这些值，或添加`useMemo()`去记忆化一个中间计算。您也不能在效果中很容易地引用这些值，因为它们无法在闭包中捕获它们。有一些约定的方法可以解决这些问题，但它们要求您在精神上"匹配"输入和输出。这很棘手，违反了 React 在其他方面的直白风格。

在 Hooks 之间传递值是我们提案的核心。渲染 props 模式是你在没有 Hooks 的情况下最接近它的方法，但是如果没有类似[Component Component](https://ui.reach.tech/component-component)的东西，你就无法获得完全的好处。这是由于"错误的层次结构"的存在，导致了大量的句法噪声。Hooks 将层次结构顺平，用于传递值，- 而函数调用是实现这一点的最简单方法。

### 缺陷 #8: 太多仪式

有许多提案属于这一'保护伞'之下。大多数都试图避免 Hooks 对 react 的依赖。有很多种方法可以做到这一点:通过内置 Hooks`this`，使他们成为一个额外的参数，你必须传递一切等。

我认为[Sebastian’s 回答](https://github.com/reactjs/rfcs/pull/68#issuecomment-439314884)比我更好地描述解决这个问题，所以我鼓励您查看它的第一部分("注入模型").

我只想说，程序员倾向于`try`/`catch`用于错误处理，在每个函数传递错误代码。这也是为什么我们更喜欢 ES 模块`import`(或 CommonJS 的`require`) 多过 AMD 的"明确"定义，这里的`require`是传给我们的。

```js
// 谁会要 AMD?
define(['require', 'dependency1', 'dependency2'], function (require) {
  var dependency1 = require('dependency1'),
  var dependency2 = require('dependency2');
  return function () {};
});
```

是的，AMD 可能更"诚实"地承认事实，但模块并没有在浏览器环境中同步加载。若一旦你了解了这一点，`define`三明治就会成为一种无知的苦差事。。

`try` / `catch`，`require`和 React-Context-api 是一个实用主义的例子，说明了我们如何希望有一些"环境"处理程序可供使用，而不是通过每个级别显式线程化 —— 即便我们通常重视明确性。我认为 Hooks 也这样的。

类似的，当我们定义组件时，我们只要抓住`React`的`Component`。 如果我们为每个组件导出一个工厂，那么我们的代码可能会与 react 脱钩:

```js
function createModal(React) {
  return class Modal extends React.Component {
    // ...
  }
}
```

但在实践中，这最终只是个恼人的间接做法。当我们真的想要其他东西握住 React 时，我们可以常在模块系统级别上这样做。

这同样适用于 Hooks。尽管如此, [Sebastian’s 回答](https://github.com/reactjs/rfcs/pull/68#issuecomment-439314884)也提到，Hooks 它是*技术上可以*从`react`"重定向"到另一个实现。([我之前文章](/how-does-setstate-know-what-to-do/)有提到过。

另一种加强仪式的方法是制作 Hooks[monadic](https://paulgray.net/an-alternative-design-for-hooks/)或者添加一个一等类`React.createHook()`。 除了运行时开销之外，任何添加包装器的解决方案，都会失去使用普通函数的巨大好处:_它们就像调试一样容易_。

普通函数允许您使用调试器进入和退出，中间没有任何库代码，并且可以准确地查看值，是如何在组件体内的流动情况。间接做法使这变得很困难。与高级组件("装饰"Hooks)或渲染 props(例如`adopt`提案或`yield`(来自发电机)也有同样的问题。间接寻址也会使静态类型复杂化。

---

正如我前面提到的，这篇文章的目的并不是要详尽。会存在不同提案还有其他有趣的问题。其中一些比较模糊(例如，与并发性或高级编译技术相关)，将来可能成为另一篇博文的主题。

Hooks 也不完美，但这是我们为解决这些问题所能找到的最佳折衷办法。我们有些事情[仍需要修复](https://github.com/reactjs/rfcs/pull/68#issuecomment-440780509)，并且存在使用Hook而不是类更尴尬的东西。这也是另一篇博客文章的主题.

无论我是否涵盖了你最喜欢的备选方案，我希望这篇文章能够帮助你们了解，我们的思考过程和选择 API 时考虑的标准。正如您所看到的，其中很多(例如确保复制粘贴、移动代码、添加和删除依赖项按预期工作)都与[优化更改有关](/optimized-for-change/)。 我希望 React 用户能够欣赏这些方面。
