> 原文：[In-depth explanation of state and props update in React](https://indepth.dev/posts/1009/in-depth-explanation-of-state-and-props-update-in-react)

在我之前的文章[【译】深入研究 React 的 Fiber 协调算法](https://github.com/shujer/Blog/issues/1)中，我已经介绍了更新过程的一些技术细节，为理解本篇文章打下了基础。

我概述了一些主要的数据结构和概念，特别是 `Fiber nodes`，`current`、`work-in-progress tree`，`side-effects` 和 `effects list`。我也概括了主要的算法，并分别解释了 `render` 和 `commit` 两个阶段的工作。如果你还没阅读过，我建议你先阅读先前的文章。

我也提供了一个用于示例的应用，它有一个按钮，可以点击增加计数并渲染到屏幕：

![](https://images.indepth.dev/images/2019/08/tmp4.gif)
你可以在[这里](https://stackblitz.com/edit/react-jwqn64)在线运行。它在 React 中实现为一个简单的组件，会在 `render` 方法返回 `button` 和 `span` 两个子元素。当你点击按钮的时候，事件处理器内部会触发组件状态的更新，最终导致 `span` 元素内部的文本更新：

```jsx
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

  componentDidUpdate() {}

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

我还添加了 `componentDidUpdate` 的生命周期方法。这里我们需要理解，React 在 `commit` 阶段是如何添加 [effects](https://github.com/shujer/Blog/issues/1) 并调用这个生命周期的。

在这篇文章，我想要向你们说明 React 如何处理状态更新和构建 `effects list` 的。我们将了解在 `render` 和 `commit` 阶段中一些主要函数中做的事情。

特别地，我们将了解 [completeWork](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/react-reconciler/src/ReactFiberCompleteWork.js#L532) 函数中会处理以下这些工作：

- 更新 `ClickCounter` 的 `count` 状态
- 调用 `render` 方法去返回子项和执行比较
- 更新 `span` 元素的 props

而 [commitRoot](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L523) 中则会：

- 更新 `span` 元素的 `textContent` 属性
- 调用 `componentDidUpdate` 生命周期方法

但在这之前，我们先了解一下，在点击事件中调用 `setState` 的时候，React 是如何调度这些工作的。

> 注意：你无需知道 React 是如何使用的，这篇文章是解释 React 内部是如何工作的。

## Scheduling updates

当我们点击按钮的时候，`click` 事件触发，之后 React 会执行我们通过 props 传入 button 的回调函数。在我们的示例中，它只是增加了计数然后更新状态：

```js
class ClickCounter extends React.Component {
    ...
    handleClick() {
        this.setState((state) => {
            return {count: state.count + 1};
        });
    }
}
```

每个 React 组件都有其相对应的 `updater`，它充当了组件和 React core 之间的一个桥梁。有了这个，才能允许 `setState` 方法在不同环境分别实现对应的逻辑，比如 ReactDOM，React Native，server side rendering（服务端渲染），testing utilities（测试工具）。

在这篇文章中，我们会探究 ReactDOM 中 `updater` 这个对象的实现，它是基于 Fiber reconciler 的。具体到 `ClickCounter` 组件，它实际是 [classComponentUpdater](https://github.com/facebook/react/blob/6938dcaacbffb901df27782b7821836961a5b68d/packages/react-reconciler/src/ReactFiberClassComponent.js#L186)。它的主要作用是创建 Fiber 的实例、将更新加入队列和调度工作。

当一个更新被加入队列的时候，它实际是将其加入到对应的 Fiber 节点的更新队列上。在我们的示例中，`ClickCounter` 组件对应的 [Fiber 节点](https://github.com/shujer/Blog/issues/1) 会有如下结构：

```js
{
    stateNode: new ClickCounter,
    type: ClickCounter,
    updateQueue: {
         baseState: {count: 0}
         firstUpdate: {
             next: {
                 payload: (state) => { return {count: state.count + 1} }
             }
         },
         ...
     },
     ...
}
```

正如你所看到的，`updateQueue.firstUpdate.next.payload` 是 `ClickCounter` 组件中传给 `setState` 的函数。它表示在 `render` 节点需要处理的第一个更新。

### Processing updates for the ClickCounter Fiber node

在之前的文章的[wook loop 章节](https://github.com/shujer/Blog/issues/1) 我解释了全局变量 `nextUnitOfWork` 的作用。它其实指向了 `workInProgress tree` 中还有工作需要处理的 Fiber 节点。尤其，当遍历 Fiber 树的时候，根据这个变量，React 就可以知道哪些 Fiber 节点还有未完成的工作。

让我们假设 `setState` 方法已经被调用。React 会把 `setState` 中的回调加入 `ClickCounter` 对应的 fiber 节点中的 `updateQueue`，然后调度工作。之后，React 进入 `render` 阶段，它从顶层的 `HostRoot` fiber 节点出发，使用 [renderRoot](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L1132) 开始遍历。它会跳过（bails out）所有已经处理的节点，直到找到未完成工作的节点。在示例中，这里只有 `ClickCounter` 这个 fiber 节点需要处理。

所有的工作都是在 Fiber 副本节点上执行的，fiber 节点上的 `alternate` 属性指向这个副本。如果在处理更新之前这个 `alternate` 节点还未创建，React 会先使用 [createWorkInProgress](https://github.com/facebook/react/blob/769b1f270e1251d9dbdce0fcbd9e92e502d059b8/packages/react-reconciler/src/ReactFiber.js#L326) 函数来创建一个副本。这里，我们先假设 `nextUnitOfWork` 保留了指向 `ClickCounter` fiber 副本节点的一个引用。

#### beginWork

首先，Fiber 会从 [beginWork](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/react-reconciler/src/ReactFiberBeginWork.js#L1489) 函数开始。

> 对于 Fiber 树中的所有节点都会执行这个函数，因此，如果你需要调试 `render` 阶段，可以在这里打断点。我经常在这里检查 Fiber 节点的类型然后找到我需要的那个。

`beginWork` 函数内部其实是一个大的 `switch` 语句，它通过 `tag` 找到 fiber 节点对应需要执行的工作类型，然后执行相对应的函数。在 `CountClick` 是一个类组件，因此，它匹配的分支如下：

```js
function beginWork(current$$1, workInProgress, ...) {
    ...
    switch (workInProgress.tag) {
        ...
        case FunctionalComponent: {...}
        case ClassComponent:
        {
            ...
            return updateClassComponent(current$$1, workInProgress, ...);
        }
        case HostComponent: {...}
        case ...
}
```

在这之后，我们进入 [updateClassComponent](https://github.com/facebook/react/blob/1034e26fe5e42ba07492a736da7bdf5bf2108bc6/packages/react-reconciler/src/ReactFiberBeginWork.js#L428) 函数。根据这个节点是否是首次渲染，React 会决定是要继续更新工作，或者创建一个实例并挂载。

```js
function updateClassComponent(current, workInProgress, Component, ...) {
    ...
    const instance = workInProgress.stateNode;
    let shouldUpdate;
    if (instance === null) {
        ...
        // 类实例不存在需要先创建实例
        constructClassInstance(workInProgress, Component, ...);
        mountClassInstance(workInProgress, Component, ...);
        shouldUpdate = true;
    } else if (current === null) {
        // 在初次渲染, 我们可以复用已有的实例
        shouldUpdate = resumeMountClassInstance(workInProgress, Component, ...);
    } else {
        shouldUpdate = updateClassInstance(current, workInProgress, ...);
    }
    return finishClassComponent(current, workInProgress, Component, shouldUpdate, ...);
}
```

#### Processing updates for the ClickCounter Fiber

我们已经有了 `ClickCounter` 组件的实例，接下来，我们进入 [updateClassInstance](https://github.com/facebook/react/blob/6938dcaacbffb901df27782b7821836961a5b68d/packages/react-reconciler/src/ReactFiberClassComponent.js#L976) 方法。React 对类组件的大部分工作都是在这里执行的。按照函数内部的执行的顺序，几个最重要的操作如下：

- 调用 `UNSAFE_componentWillReceiveProps()` （deprecated）
- 处理 `updateQueue` 中的更新，生成新的状态
- 基于新的状态调用 `getDerivedStateFromProps` 并得到执行的结果
- 调用 `shouldComponentUpdate` 确保组件想要更新；如果返回 `false` 跳过整个渲染流程，包括调用整个组件及其子项的 `render` 方法
- 调用 `UNSAFE_componentWillUpdate` （deprecated）
- 把 `componentDidUpdate` 添加到 effect 列表

> 虽然这里已经添加了用于调用 componentDidUpdate 的 effect，但是这个方法会在接下来的 commit 阶段才被执行

- 更新组件实例的 `state` 和 `props`

_组件实例的 `state` 和 `props` 必须在调用 `render` 方法之前更新，因为 `render` 的结果是基于 `state` 和 `props` 的。如果不这样做，每次都会返回相同的输出结果。_

这个函数的简化版本如下：

```js
function updateClassInstance(current, workInProgress, ctor, newProps, ...) {
    const instance = workInProgress.stateNode;

    const oldProps = workInProgress.memoizedProps;
    instance.props = oldProps;
    if (oldProps !== newProps) {
        callComponentWillReceiveProps(workInProgress, instance, newProps, ...);
    }

    let updateQueue = workInProgress.updateQueue;
    if (updateQueue !== null) {
        processUpdateQueue(workInProgress, updateQueue, ...);
        newState = workInProgress.memoizedState;
    }

    applyDerivedStateFromProps(workInProgress, ...);
    newState = workInProgress.memoizedState;

    const shouldUpdate = checkShouldComponentUpdate(workInProgress, ctor, ...);
    if (shouldUpdate) {
        instance.componentWillUpdate(newProps, newState, nextContext);
        workInProgress.effectTag |= Update;
        workInProgress.effectTag |= Snapshot;
    }

    instance.props = newProps;
    instance.state = newState;

    return shouldUpdate;
}
```

在上面的代码片段中我删除了一些辅助的代码。比如，在调用生命周期或者添加 effect 之前，React 会先使用 `typeof` 检查组件是否实现了对应的方法。例如下面的例子展示了 React 在添加 effect 前是如何检查是否有 `componentDidUpdate` 方法的：

```js
if (typeof instance.componentDidUpdate === "function") {
  workInProgress.effectTag |= Update;
}
```

到这里，我们已经知道在 render 阶段，`ClickCounter` Fiber 节点上进行了哪些操作。接下来，我们看下这些操作影响到了节点上的哪些属性的值。在 React 开始处理更新工作之前，这个 fiber 节点如下：

```js
{
    effectTag: 0,
    elementType: class ClickCounter,
    firstEffect: null,
    memoizedState: {count: 0},
    type: class ClickCounter,
    stateNode: {
        state: {count: 0}
    },
    updateQueue: {
        baseState: {count: 0},
        firstUpdate: {
            next: {
                payload: (state, props) => {…}
            }
        },
        ...
    }
}
```

在这个阶段结束后，我们得到 fiber 节点：

```js
{
    effectTag: 4,
    elementType: class ClickCounter,
    firstEffect: null,
    memoizedState: {count: 1},
    type: class ClickCounter,
    stateNode: {
        state: {count: 1}
    },
    updateQueue: {
        baseState: {count: 1},
        firstUpdate: null,
        ...
    }
}
```

**比较一下这些属性值有什么差别。**

在更新被应用之后，`memoizedState` 和 `updateQueue` 中的 `baseState` 上的 `count` 属性被更新为 `1`。React 同时也更新了 `ClickCounter` 组件实例上的状态。

执行结束之后，更新队列上没有需要执行的更新，所以 `firstUpdate` 为 `null`。还有一个重要的变化是，我们改变了 `effectTag` 的值。它从 `0` 变为 `4`。使用二进制表示是 `100`，意味着第三位被置为 1，对应了 [side-effect tag](https://github.com/facebook/react/blob/b87aabdfe1b7461e7331abb3601d9e6bb27544bc/packages/shared/ReactSideEffectTags.js) 中的 `Update` 操作：

```js
export const Update = 0b00000000100;
```

我们总结一下，当处理父节点 `ClickCounter` 的时候，React 调用了需要在更新前执行的生命周期方法，更新状态并定义相关的 side-effects。

#### Reconciling children for the ClickCounter Fiber

当父节点的工作执行完毕，React 会进入 [finishClassComponent](https://github.com/facebook/react/blob/340bfd9393e8173adca5380e6587e1ea1a23cefa/packages/react-reconciler/src/ReactFiberBeginWork.js#L355) 方法。这里，React 会调用实例上的 `render` 方法对组件返回的 children 应用 diffing 算法。在[官方文档](https://reactjs.org/docs/reconciliation.html#the-diffing-algorithm)中描述了这个算法的核心思想。相关的部分如下：

> 当我们比较 React DOM 中两个相同类型的元素的时候，React 会比较它们的属性，保留相同的 DOM 节点，只是更新需要改变的属性。

如果我们深入理解的话，这里实际比较的是 Fiber 节点上的 React 元素。这个过程是相当复杂的，我们先不查看一些具体的实现细节。我会把这个过程分成几个小的部分来讲解，特别关注 child 的 reconciliation 过程。

> 如果你想要了解一些代码细节，你可以查看 [reconcileChildrenArray](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactChildFiber.js#L732) 函数。

首先，有两个重要的工作我们需要理解：

- 当 React 处理子元素的 reconciliation 的时候，它会 **为 `render` 方法返回的子元素创建一个新的 Fiber 节点，或者更新已有的 Fiber 节点**。`finishClassComponent` 函数会返回当前 Fiber 节点上的第一个孩子节点的引用。它会被赋值给 `nextUnitOfWork`，并在之后的 work loop 中被处理。
- React 在处理父节点的工作的时候，会 **更新子节点上的 props**。这是基于 `render` 方法返回的 React 元素。

当 `ClickCounter` 组件的 children 已经完成了协调，`span` 上的 `pendingProps` 会更新，这和 `span` 元素上的值是相对应的：

```js
{
    stateNode: new HTMLSpanElement,
    type: "span",
    key: "2",
    memoizedProps: {children: 0},
    pendingProps: {children: 1},
    ...
}
```

在这之后，当 React 处理 `span` Fiber 节点上的工作时，`pendingProps` 的值会被拷贝到 `memoizedProps` 上，然后添加更新 DOM 的 effect。

这就是在 render 阶段，React 对 `ClickCounter` 执行的全部工作。因为 `ClickCouner` 的第一个孩子是 button，所以它会被赋值给 `nextUnitOfWork` 变量。如果不需进行任何工作，React 会处理它的兄弟节点，也就是 `span` 节点。根据[这里描述](https://github.com/shujer/Blog/issues/1)的算法，这个过程是在 `componentUnitOfWork` 函数中处理的。

### Processing updates for the Span fiber

现在 `nextUnitOfWork` 指向了 `span` fiber 节点的副本（alternate），React 会从这里开始工作。和之前处理 `ClickCounter` 类似，我们会从 [beginWork](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/react-reconciler/src/ReactFiberBeginWork.js#L1489) 这个函数开始。

因为 `span` 节点是 `HostComponent` 类型，它会进入 switch 语句的这个分支：

```js
function beginWork(current$$1, workInProgress, ...) {
    ...
    switch (workInProgress.tag) {
        case FunctionalComponent: {...}
        case ClassComponent: {...}
        case HostComponent:
          return updateHostComponent(current, workInProgress, ...);
        case ...
}
```

最后执行 [updateHostComponent](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/react-reconciler/src/ReactFiberBeginWork.js#L686) 函数。你可以比较下这个函数和处理类组件的 `updateClassComponent` 函数的区别。另外地，对于函数组件，则会调用 `updateFunctionComponet` 方法，其他的以此类推。所有相关的函数可以在 [ ReactFiberBeginWork.js](https://github.com/facebook/react/blob/1034e26fe5e42ba07492a736da7bdf5bf2108bc6/packages/react-reconciler/src/ReactFiberBeginWork.js) 这个文件中找到。

#### Reconciling children for the span fiber

在我们的例子中，`span` 节点上的 `updateHostComponent` 函数并不需要处理什么工作。

#### Completing work for the Span Fiber node

当 `beginWork` 执行完毕，这些节点会进入 `completeWork` 函数。但在这之前，React 需要更新 span 节点上的 `memoizedProps`。你或许记得，之前在处理 `ClickCounter` 组件的子节点协调过程中，React 已经更新了 `span` Fiber 节点上的 `pendingProps`：

```js
{
    stateNode: new HTMLSpanElement,
    type: "span",
    key: "2",
    memoizedProps: {children: 0},
    pendingProps: {children: 1},
    ...
}
```

因此，当 `span` fiber 的 `beginWork` 执行完毕，React 会更新 `memoizedProps`：

```js
function performUnitOfWork(workInProgress) {
    ...
    next = beginWork(current$$1, workInProgress, nextRenderExpirationTime);
    workInProgress.memoizedProps = workInProgress.pendingProps;
    ...
}
```

之后，会调用 `completeWork` 函数，这和前面的 `beginWork` 类似，这个函数内部也是一个大的 `switch` 语句：

```js
function completeWork(current, workInProgress, ...) {
    ...
    switch (workInProgress.tag) {
        case FunctionComponent: {...}
        case ClassComponent: {...}
        case HostComponent: {
            ...
            updateHostComponent(current, workInProgress, ...);
        }
        case ...
    }
}
```

因为 `span` Fiber 节点是 `HostComponent`，它会执行 [updateHostComponent](https://github.com/facebook/react/blob/cbbc2b6c4d0d8519145560bd8183ecde55168b12/packages/react-reconciler/src/ReactFiberBeginWork.js#L686) 函数。函数内部执行了以下工作：

- 准备更新 DOM
- 把 `span` fiber 节点上的更新添加到 `updateQueue`
- 添加更新 DOM 的 effect

在操作执行前，`span` 的 Fiber 节点是这样的：

```js
{
    stateNode: new HTMLSpanElement,
    type: "span",
    effectTag: 0
    updateQueue: null
    ...
}
```

当工作执行完毕，它是这样的：

```js
{
    stateNode: new HTMLSpanElement,
    type: "span",
    effectTag: 4,
    updateQueue: ["children", "1"],
    ...
}
```

这里的主要区别是 `effectTag` 和 `updateQueue` 的值。它从 `0` 变为 `4`。使用二进制表示是 `100`，意味着第三位被置为 1，对应了 [side-effect tag](https://github.com/facebook/react/blob/b87aabdfe1b7461e7331abb3601d9e6bb27544bc/packages/shared/ReactSideEffectTags.js) 中的 `Update` 操作。这是 React 在 commit 阶段唯一需要做的任务。`updateQueue` 保存了更新所需的数据。

当 React 处理完 `ClickCounter` 和它的子节点之后，`render` 阶段就执行完毕了。我们可以把这个已完成的副本（alternate）树赋值给 `FiberRoot` 上的 `finishedWork` 属性。这棵新的树需要被更新到屏幕上。这个工作会在 `render` 阶段结束后立即执行，如果当前 React 需要为浏览器让出时间，这个工作则会在之后执行。

## Effects list

在我们的例子中，因为 `span` 节点 和 `ClickCounter` 组件都有副作用需要执行，React 会将 `HostFiber` 上的 `firstEffect` 属性指向 `span` 的 Fiber 节点。

React 会在 [completeUnitOfWork](https://github.com/facebook/react/blob/d5e1bf07d086e4fc1998653331adecddcd0f5274/packages/react-reconciler/src/ReactFiberScheduler.js#L999) 这个函数中构建 `effect list`。以下是一棵需要执行更新副作用的 Fiber 树，它会更新 `span` 的文本内容，并调用 `ClickCounter` 的生命周期：

![](https://images.indepth.dev/images/2019/08/image.png)

这些 effect 表示为线性链表，如下：

![](https://images.indepth.dev/images/2019/08/image-1.png)

## Commit phase

这个阶段会从 [completeRoot](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L2306) 函数开始。在执行这个函数之前，会先把 `FiberRoot` 上的 `finishedWork` 属性置为 `null`：

```js
root.finishedWork = null;
```

不同于 `render` 阶段，`commit` 阶段的工作总是同步执行的，它可以保证更新 `HostRoot` 的安全性。

`commit` 阶段主要是处理 DOM 更新，以及调用 `componentDidUpdate` 等生命周期方法。这是通过遍历处理 `render` 阶段构建的 effects list 实现的。

对于 `span` 节点和 `ClickCounter` 组件，我们在 `render` 阶段定义了如下的 effects：

```js
{ type: ClickCounter, effectTag: 5 }
{ type: 'span', effectTag: 4 }
```

`ClickCounter` 的 effectTag 是 `5`（二进制中是 `101`），它被解释为调用类组件中的 `componentDidUpdate` 方法。最低有效位被置为 1 表示这个 Fiber 节点在 `render` 阶段已经完成了所有工作。

`span` 的 effectTag 是 `4`（二进制中是 `100`），它被解释为需要更新 DOM 上的元素节点。在这个例子中是 `span` 元素，React 会更新元素的 `textContent`。

### Applying effects

React 调用 [commitRoot](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L523) 函数来应用这些 effects，它包括了 3 个子函数：

```js
function commitRoot(root, finishedWork) {
  commitBeforeMutationLifecycles();
  commitAllHostEffects();
  root.current = finishedWork;
  commitAllLifeCycles();
}
```

每个子函数都会通过循环遍历 effects list 并检查 effect 的类型。当找到符合当前函数需要处理的类型时，就是应用这个 effect。在我们的例子中，它会更新 `span` 的文本内容，并调用 `ClickCounter` 的生命周期。

第一个函数 [commitBeforeMutationLifeCycles](https://github.com/facebook/react/blob/fefa1269e2a67fa5ef0992d5cc1d6114b7948b7e/packages/react-reconciler/src/ReactFiberCommitWork.js#L183) 会查找 [Snapshot](https://github.com/facebook/react/blob/b87aabdfe1b7461e7331abb3601d9e6bb27544bc/packages/shared/ReactSideEffectTags.js#L25) 类型的 effect 并调用 `getSnapshotBeforeUpdate` 方法。但是，在 `ClickCounter` 组件中我们并没有实现这个方法，所以 React 在 `render` 阶段不会将其加入 effect 中。也就是说，在我们的例子中，这个函数什么也不做。

### DOM updates

接下来，React 会执行下一个 [commitAllHostEffects](https://github.com/facebook/react/blob/95a313ec0b957f71798a69d8e83408f40e76765b/packages/react-reconciler/src/ReactFiberScheduler.js#L376) 函数。这里，React 会将 `span` 元素的文本内容从 `0` 改为 `1`。`ClickComponet` 不需要进行任何处理，因为类组件没有涉及到任何 DOM 更新的操作。

这个函数的主要作用是找到类型匹配的 effect 然后执行对应的操作。在我们的例子中我们需要更新 `span` 元素的文本，因此会进入这里的 `Update` 分支：

```js
function updateHostEffects() {
    switch (primaryEffectTag) {
      case Placement: {...}
      case PlacementAndUpdate: {...}
      case Update:
        {
          var current = nextEffect.alternate;
          commitWork(current, nextEffect);
          break;
        }
      case Deletion: {...}
    }
}
```

继续执行 `commitWork` 函数的话，我们最终会进入到 [updateDOMProperties](https://github.com/facebook/react/blob/8a8d973d3cc5623676a84f87af66ef9259c3937c/packages/react-dom/src/client/ReactDOMComponent.js#L326) 函数。它会拿到 `render` 阶段添加到 fiber 节点上的 `updateQueue` 数据，然后更新 `span` 元素的 `textContent` 属性：

```js
function updateDOMProperties(domElement, updatePayload, ...) {
  for (let i = 0; i < updatePayload.length; i += 2) {
    const propKey = updatePayload[i];
    const propValue = updatePayload[i + 1];
    if (propKey === STYLE) { ...}
    else if (propKey === DANGEROUSLY_SET_INNER_HTML) {...}
    else if (propKey === CHILDREN) {
      setTextContent(domElement, propValue);
    } else {...}
  }
}
```

当所有的 DOM 都更新完毕，React 会把 `finishedWork` 赋值给 `HostRoot`，也就是把副本（alternate）树赋值给当前树：

```js
root.current = finishedWork;
```

### Calling post mutation lifecycle hooks

最后需要执行的函数是 [commitAllLifeCycles](https://github.com/facebook/react/blob/d5e1bf07d086e4fc1998653331adecddcd0f5274/packages/react-reconciler/src/ReactFiberScheduler.js#L479)。这里 React 会调用所有需要更新后处理的生命周期方法。在 `render` 阶段，React 为 `ClickCounter` 组件添加了 `Update` effect。这是 `commitAllLifeCycles` 需要处理的一种 effect，也就是 `componentDidUpdate` 方法：

```js
function commitAllLifeCycles(finishedRoot, ...) {
    while (nextEffect !== null) {
        const effectTag = nextEffect.effectTag;

        if (effectTag & (Update | Callback)) {
            const current = nextEffect.alternate;
            commitLifeCycles(finishedRoot, current, nextEffect, ...);
        }

        if (effectTag & Ref) {
            commitAttachRef(nextEffect);
        }

        nextEffect = nextEffect.nextEffect;
    }
}
```

这个函数同时也会更新[refs](https://reactjs.org/docs/refs-and-the-dom.html)。但这里我们无需用到这个功能。我们只调用了 [commitLifeCycles](https://github.com/facebook/react/blob/e58ecda9a2381735f2c326ee99a1ffa6486321ab/packages/react-reconciler/src/ReactFiberCommitWork.js#L351) 函数：

```js
function commitLifeCycles(finishedRoot, current, ...) {
  ...
  switch (finishedWork.tag) {
    case FunctionComponent: {...}
    case ClassComponent: {
      const instance = finishedWork.stateNode;
      if (finishedWork.effectTag & Update) {
        if (current === null) {
          instance.componentDidMount();
        } else {
          ...
          instance.componentDidUpdate(prevProps, prevState, ...);
        }
      }
    }
    case HostComponent: {...}
    case ...
}
```

你可以看到，如果这是第一次渲染，React 还会调用组件的 `componentDidMount` 生命周期方法。
