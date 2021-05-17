# 深入研究 React 的 Fiber 协调算法

> 原文：[Inside Fiber: in-depth overview of the new reconciliation algorithm in React](https://indepth.dev/posts/1008/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react)

React 是一个用于构建用户界面的 JavaScript 库。它的[核心工作](https://indepth.dev/posts/1064/what-every-front-end-developer-should-know-about-change-detection-in-angular-and-react)是追踪组件的状态然后更新状态到用户界面。在 React 中我们把这个过程称为“reconciliation（协调）”。我们调用 `setState` 方法，React 会检查组件的 `state` 或者 `props` 是否更新，然后根据是否变化来决定是否重新渲染组件到界面。

React 的官方文档描述了这个架构的[核心概念](https://reactjs.org/docs/reconciliation.html)，包括：React 元素的作用，生命周期方法，`render` 方法和应用于组件子元素的 diffing 算法等。通过 `render` 方法会返回不可变的 React 元素树，也就是我们常说的“Virtual DOM”。这个术语在前期有助于人们理解 React，但这也会引起歧义，所以 React 官方文档不再使用这个术语解释。在这篇文章中，我也会始终称它为 React 元素树。

除了 React 元素树，框架也需要有一个内部实例树（实例指组件或者 DOM 元素）用于保存状态。从 React 16 开始，React 推出了内部实例树的新实现方法和用于管理实例的 **Fiber** 算法。想要了解 Fiber 架构的优点，推荐阅读这篇文章[The how and why on React’s usage of linked list in Fiber](https://indepth.dev/posts/1007/the-how-and-why-on-reacts-usage-of-linked-list-in-fiber-to-walk-the-components-tree)。

这是帮助人们理解 React 内部架构系列文章的第一篇。在这篇文章中，我会深入地解释一些重要的概念和算法相关的数据结构。当我们理解了这些之后，我们会深入研究算法以及用于遍历和处理 fiber tree 的主要函数。这个系列的下一篇文章将会展示 React 是如何使用这个算法去执行首次渲染以及处理 state 和 props 的更新的。之后，我们会探索 React 的调度（scheduler）、子元素的协调过程和构建 effects list 的机制。

在这里，我可能会提到一些高级的概念。我鼓励你阅读下面的文章，了解 Concurrent React 内部执行机制的魅力。如果你想要给 React 做开源贡献，那么这个系列的文章将是一个非常好的指南。你不需要知道如何使用 React，因为这篇文章是关于 React 内部是如何工作的。

## 背景

下面这个是一个简单的程序：
![](https://images.indepth.dev/images/2019/07/tmp1.gif)

我们有一个按钮，点击数字自增并渲染到屏幕上，实现如下：

```js
class ClickCounter extends React.Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };
    this.handleClick = this.handleClick.bind(this);
  }

  handleClick() {
    this.setState((state) => {
      return { count: state.count + 1 };
    });
  }

  render() {
    return [
      <button key="1" onClick={this.handleClick}>
        Update counter
      </button>,
      <span key="2">{this.state.count}</span>,
    ];
  }
}
```

你可以在线看这个[例子](https://stackblitz.com/edit/react-t4rdmh)。正如你所看到的，这是一个简单的组件，render 方法返回 button 和 span 两个孩子节点。只要你点击按钮，组件内部的状态更新，然后，span 元素内部的文字也会更新。

在 **协调** 过程中，React 执行了大量的工作。比如，在这个例子中，React 在首次渲染和后续的状态更新过程中，会发生以下操作：

- 更新 `ClickCounter` 组件内部状态的 `count` 属性
- 检索和比较 ClickCouter 组件的孩子节点和它们的 `props`
- 更新 `span` 元素的属性

除此之外，协调过程中还会执行其他工作，比如调用[生命周期方法](https://reactjs.org/docs/react-component.html#updating)或者更新 [refs](https://reactjs.org/docs/refs-and-the-dom.html)。这些工作只在 Fiber 架构中被统称为 “works”。根据 React 元素类型的不同会执行不同类型的工作。例如，对于类组件，React 需要创建一个实例，函数组件则不需要。正如你所知的，React 中有非常多类型的元素，比如，类组件和函数组件，宿主组件（host components），传送门（portals）等。这些 React 元素的类型是 [createElement](https://github.com/facebook/react/blob/b87aabdfe1b7461e7331abb3601d9e6bb27544bc/packages/react/src/ReactElement.js#L171) 函数的第一个参数。这个函数通常用于 `render` 方法来创建一个元素。

到这里，我们大概解释了 React 的内部工作和最主要的 fiber 算法。接下来，我们来了解下 React 内部的一些数据结构。

## 从 React 元素到 Fiber 节点

在 React 中的每个组件都有一个 UI 表示，我们称其为视图或者模板。下面是 ClickCounter 组件的模板：

```jsx
<button key="1" onClick={this.onClick}>Update counter</button>
<span key="2">{this.state.count}</span>
```

### React Elements

当在这个模板经过 JSX 编译器处理之后，你会得到一系列的 React 元素。这是 React 组件的 render 方法真正返回的内容，而不是 HTML。不过，我们并不强制要求使用 JSX，ClickCounter 的 render 方法返回的内容也可以写成这样：

```js
class ClickCounter {
    ...
    render() {
        return [
            React.createElement(
                'button',
                {
                    key: '1',
                    onClick: this.onClick
                },
                'Update counter'
            ),
            React.createElement(
                'span',
                {
                    key: '2'
                },
                this.state.count
            )
        ]
    }
}
```

`render` 方法中调用的 `React.createElement` 函数会创建并返回以下数据结构：

```js
[
    {
        $$typeof: Symbol(react.element),
        type: 'button',
        key: "1",
        props: {
            children: 'Update counter',
            onClick: () => { ... }
        }
    },
    {
        $$typeof: Symbol(react.element),
        type: 'span',
        key: "2",
        props: {
            children: 0
        }
    }
]
```

React 为每个 React 元素添加了一个内部属性 [`$$typeof`](https://overreacted.io/why-do-react-elements-have-typeof-property/) 去唯一表示每个 React 元素。除此之外，我们还使用 `type`、`key` 和 `props` 来定义元素。这些值取决于你传入 `React.createElement` 函数的值。这里，请注意 React 如何将文本内容表示为 `span` 和 `button` 节点的孩子，以及点击事件如何作为 `button` 元素 props 的一部分。React 元素还有其他的属性，比如 `ref` 等，在这篇文章不会讨论。

`ClickCounter` 的 React 元素没有任何 props 或者 key：

```js
{
    $$typeof: Symbol(react.element),
    key: null,
    props: {},
    ref: null,
    type: ClickCounter
}
```

### Fiber nodes

在协调阶段，每次 `render` 返回的 React 元素都会被合并到 fiber 树上。每个 React 元素都有对应的 fiber 节点。不同于 React 元素，fiber 节点并不是每次 `render` 都重新创建的。它们是用于保存组件状态和 DOM 的可变的数据结构。

我们上面讲到，React 更加不同类型的 React 元素会执行不同的工作。在我们的案例中，对于类组件 `ClickCounter`，它会调用生命周期和 `render` 函数，而 `span` 宿主元素（host component，DOM 节点）则执行 DOM 更新。因此，每个 React 元素都会根据它们[对应的类型](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/shared/ReactWorkTags.js) 转换为对应的 fiber 节点。

**fiber 作为一个数据结构，它能够表示一系列即将执行的工作，称为单元工作。Fiber 架构提供了一个便捷的方法去跟踪、调度、暂停和中止工作。**

当 React 元素第一次转换为 fiiber 节点，React 的 [createFiberFromTypeAndProps](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L414) 函数会使用元素中数据来创建一个 fiber 节点。在后续的更新中，React 则会复用 fiber 节点，根据对应 React 元素来更新必要的属性。React 可能还需要根据 `key` 来在同一层次中移动节点、或者删除不再使用的 React 元素对应的 fiber 节点。

> 阅读 [ChildReconciler](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactChildFiber.js#L239) 函数来查看所有的工作和 React 对于已有的 fiber 节点会执行哪些处理。

由于 React 为每个 React 元素创建对应的 fiber，因此我们将会构建一个 fiber tree。在上面的例子中，将会创建如下的结构：
![img](https://images.indepth.dev/images/2019/07/image-51.png)

所有的 fiber 节点都会基于链表使用 `child`、`sibling` 和 `return` 属性连结起来。想要了解为什么使用这个方法工作，可以查看我的另一篇文章 [The how and why on React’s usage of linked list in Fiber](https://medium.com/dailyjs/the-how-and-why-on-reacts-usage-of-linked-list-in-fiber-67f1014d0eb7)。

### Current and work in progress trees

在首次渲染之后，React 会得到一个 fiber tree，反映了用于渲染 UI 的应用状态。它通常被称为 **current tree**。当 React 开始处理更新工作的时候，会创建一个 `workInProgress tree` 用于表示即将需要更新到屏幕的状态。

fiber 中所有需要执行的工作都来自 `workInProgress tree`。当 React 遍历 `current tree` 的时候，每个现有的 fiber 节点都会创建一个候补（alternate）的节点用于创建 `workInProgress tree`（候补树）。这个节点是根据 `render` 方法返回的 React 元素的数据创建的。只要所有的更新都被处理以及所有相关的工作完成，React 会得到一个用于更新到视图的 `workInProgress tree`。当 `workInProgress tree` 被更新到视图之后，它就变成了 `current tree`

React 的一个核心概念就是一致性（consistency）。也就是说 React 总是一次性更新 DOM，而不展示部分结果。`workInProgress tree` 就像是一个用户不可见的“草稿”，基于它，React 才能处理完所有组件之后，才把对应的变化更新到屏幕上。

在源码中你会看到大量函数从 `current` 和 `workInProgress` 中获取 fiber 节点。例如，一个函数签名如下：

```js
function updateHostComponent(current, workInProgress, renderExpirationTime) {...}
```

每个 fiber 节点都有一个 **altrtnate** 指针指向候补树上对应的节点。`current tree` 上的节点会保存指向 `workInProgress tree` 上对应节点的引用。反之亦然。

### Side-effects

我们可以把 React 中的组件当成一个函数，它使用 state 和 props 去计算 UI 展示。所有其他的工作，比如更新 DOM 或者调用生命周期，都被当做一个 side-effect（副作用），或者简单称为 effect。effect 的相关概念可以在[文档](https://reactjs.org/docs/hooks-overview.html#%EF%B8%8F-effect-hook)也提到。

> 在 React 组件中，你可能会执行数据请求，订阅或者手动更改 React 组件中的 DOM。我们称这些操作为 “side effects”（或者简称 “effects”）因为它们影响其他组件而且在渲染过程中无法完成。

你可以看到，大部分 state 和 props 更新是如何导致 side-effects。既然应用 effect 是一种类型的工作，那么 fiber 节点将是一个便捷的机制去追踪更新之外的 effect。每个 fiber 节点都有向关联的 effects，通过 `effectTag` 属性表示。

因此，Fiber 中的 effects 其实定义了当所有实例被更新后需要执行的[工作](https://github.com/facebook/react/blob/b87aabdfe1b7461e7331abb3601d9e6bb27544bc/packages/shared/ReactSideEffectTags.js)。对于宿主组件（DOM 元素）来说，这些工作通常包括添加、更新或者删除元素。对于类组件来说，React 可能需要更新 `refs` 或者调用 `componentDidMount` 和 `componentDidUpdate` 生命周期方法。对于其他类型的组件也有相应的其他 effects。

### Effect list

React 需要快速处理更新，为了实现高性能，React 使用了一些有趣的技巧。其中一个是构建一个线性链表用于存储 fiber 节点的 effects，用于快速迭代。遍历一个线性链表比遍历树要快得多，同时我们并不需要处理没有 side-effects 的节点。

这个列表的目标是标记需要更新 DOM 或者执行其他副作用的节点。这个列表是 `finishedWork` 的一个子集，并通过 `nextEffect` 指针连结。

[Dan Abramov](https://medium.com/@dan_abramov) 提供了一个 effects list 的类比：他把这个 list 想象成一棵圣诞树，所有带副作用的节点都被绑上一个“圣诞灯”。为了形象化，我们可以把下面的树想象成 fiber 节点树，其中高亮的节点是需要执行某些副作用。例如，当更新导致了 `c2` 需要被插入到 DOM，`d2` 和 `c1` 需要更新属性，以及 `b2` 需要调用生命周期的方法。这些副作用会被连结起来，因此 React 可以在后面跳过其他节点。
![](https://images.indepth.dev/images/2019/07/image-52.png)

你可以看到带有 effects 的节点是如何连结起来的。当遍历节点的时候，React 使用 `firstEffect` 指针指向列表的起点。线性链表可以表示为：
![](https://images.indepth.dev/images/2019/07/image-53.png)

### Root of the fiber tree

每个 React 应用可能有一个或者多个 DOM 元素作为容器（container）。在我们的例子中，`div` 是一个带 ID 的容器。

```js
const domContainer = document.querySelector("#container");
ReactDOM.render(React.createElement(ClickCounter), domContainer);
```

React 会为每个容器创建一个[fiber root](https://github.com/facebook/react/blob/0dc0ddc1ef5f90fe48b58f1a1ba753757961fc74/packages/react-reconciler/src/ReactFiberRoot.js#L31) 对象。你能通过引用访问对应的 DOM 元素：

```js
const fiberRoot = query("#container")._reactRootContainer._internalRoot;
```

React 通过 fiber root 保存了对 fiber tree 的引用。这个引用保存在 fiber root 的 `current` 属性中。

```js
const hostRootFiberNode = fiberRoot.current;
```

fiber tree 树的根节点是一个[特殊类型](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/shared/ReactWorkTags.js#L34)的 fiber 节点 —— `HostRoot`。它是在内部创建的，并作为你最顶层的组件的父节点。`HostRoot` 节点通过 `stateNode` 属性指向 `FiberRoot` 节点：

```js
fiberRoot.current.stateNode === fiberRoot; // true
```

你可以通过 fiber root 访问到最顶层的 `HostRoot`，从而访问整个 fiber tree。或者你可以通过组件实例拿到单个 fiber 节点，例如：

```js
compInstance._reactInternalFiber;
```

### Fiber node structure

我们来看看为 `ClickCounter` 组件创建的 fiber 节点的结构：

```js
{
    stateNode: new ClickCounter,
    type: ClickCounter,
    alternate: null,
    key: null,
    updateQueue: null,
    memoizedState: {count: 0},
    pendingProps: {},
    memoizedProps: {},
    tag: 1,
    effectTag: 0,
    nextEffect: null
}
```

`span` DOM 元素则是：

```js
{
    stateNode: new HTMLSpanElement,
    type: "span",
    alternate: null,
    key: "2",
    updateQueue: null,
    memoizedState: null,
    pendingProps: {children: 0},
    memoizedProps: {children: 0},
    tag: 5,
    effectTag: 0,
    nextEffect: null
}
```

fiber 节点实际上有相当多的属性，在前面的章节我们已经解释过 `alternate`，`effectTag`， `nextEffect`，现在我们来看看其他属性的作用。

#### stateNode

存储 fiber 节点关联的组件实例、DOM 节点或者其他 React 元素的应用。通常来说，这个属性是用于存储 fiber 节点关联的本地状态。

#### type

指明 fiber 节点相关的 function 或者 class。对于类组件，它指向构造函数；对于 DOM 元素，它指明 HTML 的标签类型。我经常使用整个属性去查看一个 fiber 节点关联的是哪个元素。

#### tag

定义了[fiber 节点的类型](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/shared/ReactWorkTags.js)。在协调算法中，它用于指明接下来要做哪些工作。正如前面提到的，工作的种类取决于 React 元素的类型。[createFiberFromTypeAndProps](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L414)函数将 React 元素映射为对应类型的 fiber 节点。在我们的应用中，`ClickCounter` 组件的 `tag` 属性是 `1`，表明它是一个 `ClassComponent`；`span` 元素的 `tag` 属性是 `5` 表明它是一个 `HostComponent`。

#### updateQueue

一个队列，用于状态更新、回调函数或者 DOM 更新。

#### memoizedState

用于创建输出的 fiber 节点状态。当处理更新的时候，它反映了当前渲染到屏幕的状态。

#### memoizedProps

fiber 在上次渲染的期间用于创建输出的 props

#### pendingProps

从新更新的 React 元素获取的 props，将被应用于子组件或者 DOM 元素

#### key

在一组 children 中作为唯一的标识，帮助 React 找到哪些子项发生了改变，被添加到列表或者从列表中移除。它和 React 在[这里](https://reactjs.org/docs/lists-and-keys.html#keys)描述的 “lists and keys” 功能相关。

你可以在[这里](https://github.com/facebook/react/blob/6e4f7c788603dac7fccd227a4852c110b072fe16/packages/react-reconciler/src/ReactFiber.js#L78)看到 fiber 节点的完整数据结构。我没有全部解释这些属性。其中，用于构建整个树的 `child`，`sibling` 和 `return` 属性我已经在[我之前的文章](https://indepth.dev/posts/1007/the-how-and-why-on-reacts-usage-of-linked-list-in-fiber-to-walk-the-components-tree)中描述了。其他的一些属性，比如 `expirationTime`，`childExpirationTime` 和 `mode` 是用于 `Scheduler` 的。

## 主要算法

React 执行工作可以分为两个主要的阶段：**render** 和 **commit**。

- 在 `render` 阶段，React 通过 `setState` 或者 `React.render` 将更新应用到被调度的组件并找到哪些需要被更新到 UI 上。如果它是首次 `render` 的话，React 会为 `render` 返回的每个元素创建一个新的 fiber 阶段。在后续更新中，现有的 React 元素对应的 fiber 节点会被复用和更新。**`render` 阶段会返回一棵节点带有 side-effects 标记的 fiber 树**。这些 effects 描述了在接下来的 `commit` 阶段需要做的工作。

- 在 `commit` 阶段，React 会拿到有 effects 标记的 fiber tree，并将它们应用到实例上。实际上，它是通过遍历 effect 链表去执行 DOM 更新或者其他用户可见的操作。

一个关键的地方是，理解 **`render` 阶段所执行的工作可以是异步的**。React 在处理 fiber 节点过程，当没有剩余时间的时候，会中止和保存现有的工作，为其他事件让出调度。之后，它会恢复并继续上次未完成的工作。有的时候，可能需要放弃已完成的工作并从顶层开始重新执行。之所以能够中断这些工作，是因为在这个阶段中进行的工作是用户不可见的，比如没有涉及到 DOM 更新的操作。与之相反的，在接下来的 **`commit` 阶段总是同步的**。这是因为在这个阶段进行的工作会导致的变化是用户可见的，比如 DOM 更新。因此，React 在这个阶段需要一次性执行完这些工作。

调用生命周期方法是 React 需要执行的一种工作。有些方法是在 `render` 阶段调用的，有些则是在 `commit` 阶段调用的。在 `render` 阶段调用的生命周期有：

- [UNSAFE_]componentWillMount (deprecated)
- [UNSAFE_]componentWillReceiveProps (deprecated)
- getDerivedStateFromProps
- shouldComponentUpdate
- [UNSAFE_]componentWillUpdate (deprecated)
- render

正如你所看到的，从 React 16.3 版本开始，一些在 `render` 阶段调用的 legacy 生命周期被标记为 `UNSAFE`。它们在官方文档中现在被称为 legacy lifecycles。在未来的 React 16.x 版本中它们可能会被废弃，而对应的没有带 `UNSAFE` 前缀标识的生命周期在 React 17.0 会被移除。你可以在[这里](https://reactjs.org/blog/2018/03/27/update-on-async-rendering.html)查看相关的变化和建议的迁移路径。

你是否好奇为什么要标记这些生命周期为 `UNSAFE` 呢？

在前面，我们已经知道 `render` 阶段不会处理诸如 DOM 更新等的副作用，因此 React 可以异步地更新组件（甚至多线程地处理）。然而，被标记为 `UNSAFE` 的生命周期总是被错误地理解或者使用。开发人员可能会在这些方法中执行带有副作用的代码，而这会引发新的异步渲染问题。尽管没有带 `UNSAFE` 前缀标识的方法将会被移除，但它们在将推出的 Concurrent Mode 模式下依然会引起问题。

在 `commit` 阶段，会执行以下的这些生命周期：

- getSnapshotBeforeUpdate
- componentDidMount
- componentDidUpdate
- componentWillUnmount

由于这些方法是在同步的 `commit` 阶段被执行的，因此它们可以包含与 DOM 相关的副作用。
现在，我们已经对 React 如何遍历树和执行工作有了一个大概的认识，接下来我们继续深入地探索。

### Render phase

协调算法总是从使用 [renderRoot](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L1132) 函数的最顶层的 `HostRoot` 节点开始。然而，React 有可能 bails out (跳过) 已经处理过的节点，直到找到还没有完成工作的节点。例如，当你在组件树的层次节点中调用 `setState`，React 会从根节点开始然后快速跳过父组件，直到找到调用了 `setState` 的组件。

#### Main steps of the work loop

所有的 fiber 节点都在一个 [work loop](https://github.com/facebook/react/blob/f765f022534958bcf49120bf23bc1aa665e8f651/packages/react-reconciler/src/ReactFiberScheduler.js#L1136) 中被处理。以下是它实现异步执行的关键代码：

```js
function workLoop(isYieldy) {
  if (!isYieldy) {
    while (nextUnitOfWork !== null) {
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    }
  } else {...}
}
```

在上面的代码中，`nextUnitOfWork` 指向 `workInProgress tree` 中需要被处理的 fiber 节点。当 React 遍历 fiber 树的时候，会使用这个遍历来检查是否还有需要处理的 fiber 节点。当当前的 fiber 被处理之后，`nextUnitOfWork` 这个变量会更新为下一个需要处理的节点或者 `null`。如果没有需要处理的几点，React 会退出 `work loop` 然后准备 commit。

在遍历树、初始化和 complete 工作的过程中，涉及到 4 个主要的函数：

- [performUnitOfWork](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L1056)
- [beginWork](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/react-reconciler/src/ReactFiberBeginWork.js#L1489)
- [completeUnitOfWork](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L879)
- [completeWork](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/react-reconciler/src/ReactFiberCompleteWork.js#L532)

以下是一个遍历 fiber 树的动画，演示了这些函数是如何被使用的。在这个例子中，我简化了函数的实现。每个函数都需要一个 fiber 节点，当 React 在向下处理树的时候，你可以看到当前正在处理的 fiber 节点。你可以清楚地看到算法是如何从两个分支之间切换的 —— 它会先完成所有子节点的处理，然后再返回父节点。

![](https://images.indepth.dev/images/2019/08/tmp2.gif)

> 注：这里竖线连接的是相邻节点（siblings）,曲线连接的是孩子节点（children）。例如，`b1` 没有孩子节点，`b2` 有一个孩子 `c1`。

你可以在这个[视频链接](https://vimeo.com/302222454)中暂停和查看当前的节点和函数的状态。从概念上来说，你可以把 “begin” 理解成在组件中 “stepping into”，“complete” 理解成 “stepping out” 组件。你可以在[这里](https://stackblitz.com/edit/js-ntqfil?file=index.js)查看这个例子的实现代码。

我们先来看看 `performUnitOfWork` 和 `beginWork` 两个函数：

```js
function performUnitOfWork(workInProgress) {
  let next = beginWork(workInProgress);
  if (next === null) {
    next = completeUnitOfWork(workInProgress);
  }
  return next;
}

function beginWork(workInProgress) {
  console.log("work performed for " + workInProgress.name);
  return workInProgress.child;
}
```

`performUnitOfWork` 函数接收 `workInProgress tree` 的 fiber 节点，然后使用 `beginWork` 开始执行工作。这个函数会处理一个 fiber 节点所需要的全部工作。为了演示现在正在进行的工作，我们先简单地打印出 fiber 节点的名字。**`beginWork` 函数总是返回需要在 loop 中处理的下一个子节点或者 `null`**。

如果有下一个子节点，它会在 `workLoop` 函数中，把 `nextUnitOfWork` 赋值为该节点。如果没有子节点，React 则知道已经到达了分支的最底层，所以完成当前节点的工作。**一个节点的工作完成后，需要执行相邻兄弟节点的工作，之后回溯到当前的父节点**。这个工作是在 `completeUnitOfWork` 函数中完成的：

```js
function completeUnitOfWork(workInProgress) {
  while (true) {
    let returnFiber = workInProgress.return;
    let siblingFiber = workInProgress.sibling;

    nextUnitOfWork = completeWork(workInProgress);

    if (siblingFiber !== null) {
      // If there is a sibling, return it
      // to perform work for this sibling
      return siblingFiber;
    } else if (returnFiber !== null) {
      // If there's no more work in this returnFiber,
      // continue the loop to complete the parent.
      workInProgress = returnFiber;
      continue;
    } else {
      // We've reached the root.
      return null;
    }
  }
}

function completeWork(workInProgress) {
  console.log("work completed for " + workInProgress.name);
  return null;
}
```

这个函数的主体是一个大的 `while` 循环。当 `workInProgress` 节点没有子节点的时候，React 会进入这个函数。当完成所有当前节点的工作，先检查是否有相邻的兄弟节点。有的话，React 会返回 sibling 节点并退出函数。`nextUnitOfWork` 会被指向该节点，然后 React 会从 sibling 节点开始另一个分支的工作。这里的关键是，React 只有完成了前面的兄弟节点之后才会处理剩余的节点。**只有所有分支的节点（包括子节点）处理完毕才算完成工作，然后回溯到父节点**。

正如你在实现中看到的，`performUnitOfWork` 和 `completeUnitOfWork` 函数主要是用于处理迭代，而 `beginWork` 和 `completeWork` 函数是处理实际的工作。在接下来的系列文章中，我们会了解在 `beginWork` 和 `completeWork` 函数中， `ClickCounter` 组件和 `span` 节点发生了什么。

### Commit phase

`commit` 阶段是从 [completeRoot](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L2306) 函数开始的。React 会开始更新 DOM 和调用前继或者后继 mutation 生命周期。

当 React 进入这个阶段的时候，我们已经拥有 2 棵树和一个 effects 链表。其中一棵树表示渲染到当前屏幕的状态，另一棵（替代）树是在 `render` 阶段创建的，它在源码被称为 `finishedWork` 或者 `workInProgress`，表示需要更新到屏幕的状态。alternate tree 和 current tree 一样，使用 `child` 和 `sibling` 指针连结到一起。

这里还有一个 effects 链表 —— 它是 `finishedWork` 树节点的一个子集，节点之间通过 `nextEffect` 指针连结。effect 链表是 `render` 阶段创建的。`render` 阶段的主要目的是知道哪些节点需要被插入、更新或者删除，以及哪些生命周期需要被调用。而这正是 effect 链表告诉我们的。**在 commit 阶段会遍历 effect 链表上的节点**。

> 为了方便调试，current tree 可以通过 fiber root 的 current 属性访问到。而 finishedWork tree 可以通过 current tree 上的 HostFiber 节点的 alternate 属性访问到。

在 commit 节点执行的主要函数是 [commitRoot](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L523)。它主要做了以下的工作：

- 对 effectTag 为 `Snapshot` 的节点，调用 `getSnapshotBeforeUpdate` 生命周期
- 对 effectTag 为 `Deletion` 的节点，调用 `componentWillUnMount` 生命周期
- 执行所有 DOM 插入、更新和删除的工作
- 把 current tree 指向 `finishedWork`
- 对 effectTag 为 `Placement` 的节点，调用 `componentDidMount` 生命周期
- 对 effectTag 为 `Update` 的节点，调用 `componentDidUpdate` 生命周期

当调用完 `getSnapshotBeforeUpdate` 生命周期之后，React 会执行树上的所有副作用。这是分成两个步骤完成的。

- 在第一个步骤中，会执行所有 DOM（host）的插入、更新删除和 ref 卸载操作。
- 之后，React 把 `finishedWork` 赋值给 `FiberRoot`，也就是把 `current` tree 指向 `workInProgress` tree。这个工作是在 commit 阶段的第一个步骤完成之后而且第二个步骤还没开始之前完成的，这样在 `componentWillUnmount` 过程中，current tree 依旧是之前的树，在 `componentDidMount/Update` 阶段 current tree 则是 finishedWork。
- 在第二个步骤，React 会调用其他生命周期和 ref 回调。这些方法是作为单独的过程来执行的，保证整个树中所有的替换、更新和删除操作都已经完成。

下面是一个函数，执行了上述描述的步骤：

```js
function commitRoot(root, finishedWork) {
  commitBeforeMutationLifecycles();
  commitAllHostEffects();
  root.current = finishedWork;
  commitAllLifeCycles();
}
```

这里每个子函数都会遍历 effects 链表并检查 effect 的类型，找到相应需要执行的函数并应用。

#### Pre-mutation lifecycle methods

例如，这是遍历 effect 链表并检查一个节点是否有 `SnapShot` 标签的代码：

```js
function commitBeforeMutationLifecycles() {
  while (nextEffect !== null) {
    const effectTag = nextEffect.effectTag;
    if (effectTag & Snapshot) {
      const current = nextEffect.alternate;
      commitBeforeMutationLifeCycles(current, nextEffect);
    }
    nextEffect = nextEffect.nextEffect;
  }
}
```

对于类组件来说，这个 effect 代表调用 `getSnapshotBeforeUpdate` 方法。

#### DOM updates

React 使用 [commitAllHostEffects](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L376) 函数来更新 DOM。这个函数主要指明了一个节点需要执行的操作的类型并执行：

```js
function commitAllHostEffects() {
    switch (primaryEffectTag) {
        case Placement: {
            commitPlacement(nextEffect);
            ...
        }
        case PlacementAndUpdate: {
            commitPlacement(nextEffect);
            commitWork(current, nextEffect);
            ...
        }
        case Update: {
            commitWork(current, nextEffect);
            ...
        }
        case Deletion: {
            commitDeletion(nextEffect);
            ...
        }
    }
}
```

有趣的是，React 会在 `commitDeletion` 函数中调用 `componentWillUnmount` 方法。

#### Post-mutation lifecycle methods

React 使用 [commitAllLifecycles](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L465) 函数来执行剩余的生命周期 `componentDidUpdate` 和 `componentDidMount`。
