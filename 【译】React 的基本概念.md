> 原文：[React - Basic Theoretical Concepts](https://github.com/reactjs/react-basic)

在这篇文章中我打算正式地讲讲我对于 React 心智模型的理解。我会采用演绎推理的描述方法，逐渐深入地讲解其中的设计。

这里有些前提可能是有争议的，而且使用的例子也可能有缺陷或者漏洞。这篇文章只是一个简单的开始。如果你有更好地方式来描述 React 的概念和理论，欢迎提交 pull request。文章会按 `简单 -> 复杂` 的顺序来讲解，不会涉及过多的框架细节。

React.js 的实现涉及很多细节，比如实用的解决方案、渐进的步骤、算法的优化、遗留代码、调试工具以及用于高性能运行的设计。有些东西可能是暂时的，会随着时间的推移改变。实际的实现是很复杂的，难以解释。

对此，我有一个简化的心智模型，便于理解。

## Transformation

理解 React 的核心概念之前，我们要认识到组件只是通过映射关系，把数据变换成另一种形式的数据（UI）。如果输入相同，那么输出一定也相同。这恰好就是纯函数：

```javascript
function NameBox(name) {
  return { fontWeight: "bold", labelContent: name };
}
```

```bash
'Sebastian Markbåge' ->
{ fontWeight: 'bold', labelContent: 'Sebastian Markbåge' };
```

## Abstraction

你很难用单个函数来实现一个复杂的组件。一个组件应该可以被抽象成多个可复用的部分，同时隐藏内部的实现细节是至关重要的。例如，通过在一个函数中调用另一个函数：

```javascript
function FancyUserBox(user) {
  return {
    borderStyle: "1px solid blue",
    childContent: ["Name: ", NameBox(user.firstName + " " + user.lastName)],
  };
}
```

```bash
{ firstName: 'Sebastian', lastName: 'Markbåge' } ->
{
  borderStyle: '1px solid blue',
  childContent: [
    'Name: ',
    { fontWeight: 'bold', labelContent: 'Sebastian Markbåge' }
  ]
};
```

## Composition

为了真正实现可复用的特性，每次只复用叶子节点并为之构建一个新的容器显然是不够的。你还需要结合已经抽象的部分进行二次组合。在我看了，“组合”是能够组合两个或者多个抽象部分，形成一个新的抽象。

```javascript
function FancyBox(children) {
  return {
    borderStyle: "1px solid blue",
    children: children,
  };
}

function UserBox(user) {
  return FancyBox(["Name: ", NameBox(user.firstName + " " + user.lastName)]);
}
```

## State

组件不仅仅只是对服务器响应的数据/业务逻辑状态的一个复制。实际上，有很多状态是用于实现特定的渲染。例如，你在一个“文本域”中打字。在打字过程中的状态我们并不需要同步到其他页面或者手机设备上。再例如，滚动位置，它也不需要被复制到在其他渲染容器上。

我们更加倾向于使用不可变的数据模型，把每个更新状态的函数当作一个原子操作，并且至上而下地串联使用。

```javascript
function FancyNameBox(user, likes, onClick) {
  return FancyBox([
    "Name: ",
    NameBox(user.firstName + " " + user.lastName),
    "Likes: ",
    LikeBox(likes),
    LikeButton(onClick),
  ]);
}

// Implementation Details

var likes = 0;
function addOneMoreLike() {
  likes++;
  rerender();
}

// Init

FancyNameBox(
  { firstName: "Sebastian", lastName: "Markbåge" },
  likes,
  addOneMoreLike
);
```

> 注：这里的例子使用副作用去更新状态。真正的心智模型是通过“更新”返回一个新版本的状态，但那样解释起来比较复杂，后面再更新这个示例。

## Memoization

如果我们已经知道一个函数是纯函数，那么多次重复地调用这个函数显然有些浪费资源。我们可以保存一个函数执行的缓存版本，用于追踪传入的参数和最后执行的结果。那么，如果下次使用相同的值就不必重复执行了。

```javascript
function memoize(fn) {
  var cachedArg;
  var cachedResult;
  return function (arg) {
    if (cachedArg === arg) {
      return cachedResult;
    }
    cachedArg = arg;
    cachedResult = fn(arg);
    return cachedResult;
  };
}

var MemoizedNameBox = memoize(NameBox);

function NameAndAgeBox(user, currentTime) {
  return FancyBox([
    "Name: ",
    MemoizedNameBox(user.firstName + " " + user.lastName),
    "Age in milliseconds: ",
    currentTime - user.dateOfBirth,
  ]);
}
```

## Lists

大部分的 UI 都是展示有多个不同数据项的列表。这形成了一个天然的层级结构。
为了管理每个数据项的状态，我们可以创建一个 Map 来管理每个特定项的状态。

```javascript
function UserList(users, likesPerUser, updateUserLikes) {
  return users.map((user) =>
    FancyNameBox(user, likesPerUser.get(user.id), () =>
      updateUserLikes(user.id, likesPerUser.get(user.id) + 1)
    )
  );
}

var likesPerUser = new Map();
function updateUserLikes(id, likeCount) {
  likesPerUser.set(id, likeCount);
  rerender();
}

UserList(data.users, likesPerUser, updateUserLikes);
```

> 注：我们现在给 FancyNameBox 函数传入了多个参数。这会破坏我们之前的缓存，因为之前的写法每次只记住一个值。后面我们会讲多个参数的情况。

## Continuations

不幸的是，由于 UI 中包含大量列表，我们需要很多样板文件来管理它们。
我们可以通过延迟函数的执行，来将一些样板文件从我们的关键业务逻辑代码中分离。例如，使用“柯里化”（JavaScript 中的 `bind`）。从函数的外部传入状态，从而在函数中不在使用样板文件。
这种方法并没有减少样板文件，只是将样本文件从我们的核心业务逻辑中剥离。

```javascript
function FancyUserList(users) {
  return FancyBox(UserList.bind(null, users));
}

const box = FancyUserList(data.users);
const resolvedChildren = box.children(likesPerUser, updateUserLikes);
const resolvedBox = {
  ...box,
  children: resolvedChildren,
};
```

## State Map

从上面我们已经知道，我们希望使用组合来避免重复编写相同的逻辑。这可以通过提取状态，然后再将状态传递到多个需要的低层函数来实现。

```javascript
function FancyBoxWithState(
  children,
  stateMap,
  updateState
) {
  return FancyBox(
    children.map(child => child.continuation(
      stateMap.get(child.key),
      updateState
    ))
  );
}

function UserList(users) {
  return users.map(user => {
    continuation: FancyNameBox.bind(null, user),
    key: user.id
  });
}

function FancyUserList(users) {
  return FancyBoxWithState.bind(null,
    UserList(users)
  );
}

const continuation = FancyUserList(data.users);
continuation(likesPerUser, updateUserLikes);
```

## Memoization Map

一旦我们想要缓存列表中的多个数据项，这显得有点困难。你需要实现一个复杂的缓存算法，以平衡内存使用率和函数调用频率。
幸运的是，UI 在相同的位置通常是比较稳定的。在一棵树中相同的位置通常会得到一个相同的值。在缓存计算中，树实际上是一种相当有用的策略。
我们可以使用处理状态时相同的方法，将缓存传递到复合函数中。

```javascript
function memoize(fn) {
  return function (arg, memoizationCache) {
    if (memoizationCache.arg === arg) {
      return memoizationCache.result;
    }
    const result = fn(arg);
    memoizationCache.arg = arg;
    memoizationCache.result = result;
    return result;
  };
}

function FancyBoxWithState(children, stateMap, updateState, memoizationCache) {
  return FancyBox(
    children.map((child) =>
      child.continuation(
        stateMap.get(child.key),
        updateState,
        memoizationCache.get(child.key)
      )
    )
  );
}

const MemoizedFancyNameBox = memoize(FancyNameBox);
```

## Algebraic Effects 代数效应

当我们需要在多个抽象层级中传递很多零散的值，是相当麻烦的。我们有时希望有一个捷径能够在两个抽象层级中传递数据，并且不影响到它们的中间节点。在 React 中，这个捷径就是“context”。
但是有的时候，两个抽象层级之间的数据流动并非严格地遵循树自上而下的关系。例如，在布局算法中，我们需要在布局之前知道子节点的大小。
我会使用我之前关于 [代数效应](https://www.eff-lang.org/) 的一个 [ECMAScript 提案](https://esdiscuss.org/topic/one-shot-delimited-continuations-with-effect-handlers) 来说明。如果你非常熟悉函数编程的话，它们是避免了 monads 引入的仪式代码。

```javascript
function ThemeBorderColorRequest() { }

function FancyBox(children) {
  const color = raise new ThemeBorderColorRequest();
  return {
    borderWidth: '1px',
    borderColor: color,
    children: children
  };
}

function BlueTheme(children) {
  return try {
    children();
  } catch effect ThemeBorderColorRequest -> [, continuation] {
    continuation('blue');
  }
}

function App(data) {
  return BlueTheme(
    FancyUserList.bind(null, data.users)
  );
}
```
