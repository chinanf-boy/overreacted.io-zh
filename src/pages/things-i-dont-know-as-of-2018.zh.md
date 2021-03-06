---
title: 自2018年以来，我不知道的事情
date: '2018-12-28'
spoiler: 我们可以承认我们的知识差距，但不会贬低我们的专业知识.
---

人们常常认为我知道的远比我实际做的要多。这不是一个坏的想法，而我也没有去抱怨什么。(来自少数群体的人们，尽管他们的资历不简单，但往往会受到相反的偏见，这样给人感觉，就很*糟糕啦*)

**在本文中，我将给出一个不完全的编程主题列表，其中大多是人们常常错误以为，我了解了这些主题。**我不是说*你*不需要学习它们 - 或者我不知道*其他*有用的东西。但由于我现在自己并不处于弱势地位，我可以坦诚地说.

这里，才是我认为很重要的.

---

首先，通常有一个不切实际的期望，一个有经验的工程师知道他们领域的每一项技术。你看过由一百个库和工具组成的"学习路线图"了吗?这很有用，但很吓人。

更重要的是，无论你有多丰富的经验，你可能仍会感觉和发现，自己在有能力、不充分("冒名顶替者综合症")和过度自信("邓宁-克鲁格效应")之间转换。这取决于你的环境、工作、个性、队友、精神状态、一天中的时间等等。

经验丰富的开发人员，有时会公开他们的不安全感，以鼓励初学者。但是一个经验丰富的外科医生仍会感到紧张，和一个学生刚拿着他们的第一把手术刀，这两者之间是有天壤之别的!

也许你听到"我们都是初级开发人员"会感到沮丧，听起来就像是，对面临实际知识缺口的读书人，的一次空谈谬论。即使，是对我这样的好好从业先生( well-intentioned practitioners)来说，啊 Q 精神(Feel-good confessions)也是无法弥合的。

尽管如此，即便是经验丰富的工程师也有许多知识缺口。这篇文章就是关于我，和去鼓励那些能承受自身脆弱的人，分享他们自己的缺口。但在我们这样做的时候，请不要贬低我们的经验.

**我们可以承认我们的知识缺口，可能会觉得或者不觉得自己是冒名顶替症者，但我们仍然拥有非常宝贵的专业知识，这需要我们多年的努力才能得到。**

---

有了这个免责声明，那这里就只是我不知道的一些事情而已：

- **Unix 命令和 bash。**我可以`ls`和`cd`，但我查了其他的一切.我知道管道的概念，但我只在简单的情况下使用它。我不知道怎么用`xargs`来创建复杂的管道链，或者如何组合和重定向不同的输出流。我也从来没有正确地学习过 bash，所以我只能编写非常简单(而且经常是错误的)shell 脚本。

- **低层语言。**我知道汇编语言可以让你在内存中存储东西，然后能在代码间跳来跳去，但就到此为止了。我写了几行 C 并理解指针是什么，但我不知道如何使用`malloc`或其他手动内存管理技术。从来没有玩 Rust。

- **网络堆栈。**我知道计算机有 IP 地址，而 DNS 是我们解析主机名的方法。我知道有像 TCP/IP 这样的低级协议来交换包，以(是吗?)确保完整性。就这样-我对细节很模糊。

- **容器。**我不知道如何使用 docker 或 kubernetes。(它们是相关?)我有一个模糊的想法，是不是他们可以让我，以一种可预测的方式玩转一个单独的虚拟机。听起来不错，但我没试过。

- **无服务器的** 听起来也很酷。从未尝试过。我不清楚这个模型是如何改变后端编程的(如果它真做到了的话).

- **微服务。**如果我理解正确的话，它说的是"许多 API 端点能互相交谈"。我不知道这种方法的实际优点或缺点是什么，因为我没有使用过它。

- **Python。**我对这个感觉很不好 - 我*有*在某种程度上与 Python 合作了几年，但从来没有真正学习过它。其中有很多事情,比如导入行为，对我来说是完全不透明的.

- **Node 后端。**我了解如何运行 Node，使用了一些 API，比如`fs`用于建立工具，并能建立 Express 框架。但我从未让一个 Node 和一个数据库进行交谈过，也不知道如何在其中编写后端。我也不熟悉像 next 这样的 React 框架，除了"你好世界"。

- **本地平台。**我试着在某个时候学习 Objective C,但没有成功。我也没学过 Swift。Java 也一样。(不过,我可能会把它捡起来,因为我和 C# 一起工作.)

- **算法。**你能从我身上得到的最多的是冒泡排序，也许在一个好日子里用快速排序。如果任务与一个特定的实际问题相关联，我可能会做简单的图遍历任务。我理解 O(N)符号，但我的理解并不比"不要在循环中，再放置循环"深多少程度。

- **函数语言。**除非你算上 javascript,否则我不会流利地使用任何传统的函数语言(我只会流利的 C# 和 javascript——我已经忘记了大部分的 C# 了.)。我很难阅读 受 Lisp 启发的 (像 Clojure)、受 Haskell 启发的(像 Elm)或 受 ML 启发的(像 Ocaml)代码。

- **函数术语。**映射(Map)和减少是熟悉的，对我而言。我不知道 monoids,functors,等等。我好像知道 monad 是什么,但也可能是幻觉.

- **现代 CSS。**我不知道 Flexbox 或 Grid。Floats 才是我的果酱.

- **CSS 方法。**我用了 BEM(意思是 css 那部分,而不是原来的 BEM)，但这就是我知道的所有。我没有尝试过 OOCSS 或其他方法.

- **SCSS/SASS** 没必要学它们.

- **CORS** 我害怕这些错误!我知道我需要设置一些 headers 来修复它们，但我过去在这里浪费了很多时间。

- **HTTPS/SSL。**从不设置。不知道它的概念是如何超越了，私钥和公钥。

- **GraphQL。**我可以阅读一个查询(query)，但我真的不知道如何用 Node 和 edges 来表达内容，何时使用片段(fragments),以及分页在那里是如何工作的.

- **Sockets。**我的理解模式是，它们让计算机在请求/响应模式之外，能互相交流,但这就是我知道的所有。

- **Streams。**除了 Rx 观测数据，我还没有与 流(Streams) 密切合作过。我使用过一到两次旧的 Node 流，但总是在处理错误。

- **Electron**从来没有尝试过.

- **TypeScript。**我理解类型的概念，可以阅读注释,但我从来没有写过。我试了几次,就遇到了困难。

- **部署和开发。**我可以通过 ftp 发送一些文件或者杀死一些进程,但这是我的 DevOps 技能的极限了。

- **绘图。**无论是 canvas、SVG、WebGL 还是低级别的图形,我都不擅长它。我知道整体的想法,但我需要学习原始的.

当然，这份清单并不详尽。有很多事情我都不知道。

---

讨论起来似乎很奇怪。写起来甚至感觉不对。我是在吹嘘我的无知吗? 我想从这篇文章中，得到的信息是:

- **即使你最喜欢的开发人员，也可能不知道你知道的很多事情。**

- **不管你的知识水平如何，你的信心都可以有很大的不同。**

- **经验丰富的开发人员，拥有宝贵的专业知识，尽管知识有缺口。**

我知道我的知识有缺口(至少，是其中一些)。如果我变得好奇或者我需要它们做一个项目，我可以稍后再填写.

这并没有贬低我的知识和经验。有很多事情我可以做得很好。例如,在我需要的时候，再学习技术。

> 更新:我也[写了](/the-elements-of-ui-engineering/)关于一些我知道的事情.
