# React - 前言

&nbsp;&nbsp;该文档是 react 开发者最初的设计蓝图，其目的在于指导开发实践。由于是设计蓝图，因此在具体开发中可能会出现一些问题或实现差异，这都无关紧要，最重要的是能够开启一个完整的设计之旅。

&nbsp;&nbsp;React.js UI 库的开发实现充满了务实的解决方案、增量迭代、算法优化，以及调试工具等完整的体系。这些工具/算法都会不断演进，在不同版本的 React 发布时使用者们将会深有体会

## 从数据到 UI

&nbsp;&nbsp;React 库的核心思想即 UI 仅仅只是数据的映射，相同的数据会产生相同的 UI 展示。与 WebComponents 规范不同，React 最主要的目的并不是解决 UI 组件的复用，而是为了 UI 和数据的绑定。

```js
function NameBox(name) {
  return { fontWeight: "bold", labelContent: name };
}
```

```
'Sebastian Markbåge' ->
{ fontWeight: 'bold', labelContent: 'Sebastian Markbåge' };
```

## 抽象

&nbsp;&nbsp;复杂的 UI 绘制很难通过单一的函数实现，并且在前端开发中也不推荐单体的 UI 控制。事实上，Web 页面本身具有层次性和可复用性，因此将复杂的 UI 拆分为若干较小的 UI 片段并且对外提供简单的调用接口。这就是对 UI 的抽象，类似入口函数调用其他功能函数来完成复杂的任务。

```js
function FancyUserBox(user) {
  return {
    borderStyle: "1px solid blue",
    childContent: ["Name: ", NameBox(user.firstName + " " + user.lastName)]
  };
}
```

```
{ firstName: 'Sebastian', lastName: 'Markbåge' } ->
{
  borderStyle: '1px solid blue',
  childContent: [
    'Name: ',
    { fontWeight: 'bold', labelContent: 'Sebastian Markbåge' }
  ]
};
```

## 组合

&nbsp;&nbsp;组合是一种软件设计方法，即通过将若干独立的抽象组合为完整的功能实体输出，从而完成更加复杂的任务.

```js
function FancyBox(children) {
  return {
    borderStyle: "1px solid blue",
    children: children
  };
}

function UserBox(user) {
  return FancyBox(["Name: ", NameBox(user.firstName + " " + user.lastName)]);
}
```

## 状态

&nbsp;&nbsp;UI 并不简单是服务器/客户端业务逻辑的复制，UI 在与用户进行交互时也会产生一些状态。这些状态作为数据源的另一种组成形式，同样会反馈到 UI 的呈现。可以认为通过单纯的外部数据来指导 UI 渲染并不能很好地提供体验效果，只有加入了状态才会产生丰富多彩的 UI 应用

```js
function FancyNameBox(user, likes, onClick) {
  return FancyBox([
    "Name: ",
    NameBox(user.firstName + " " + user.lastName),
    "Likes: ",
    LikeBox(likes),
    LikeButton(onClick)
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

注意: 上面的例子在更新 state 时存在副作用，在 react 的实现中采用了对 state 类似版本控制的方式来完成无副作用更新。

## 记忆化

&nbsp;&nbsp;对于纯函数的调用，相同的输入一定产生相同输出。为了提高调用性能，我们创建一个具有记忆能力的函数，用于记录函数调用时的最后一个参数及返回值，从而在下一次对相同参数的调用直接返回结果

```js
function memoize(fn) {
  var cachedArg;
  var cachedResult;
  return function(arg) {
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
    currentTime - user.dateOfBirth
  ]);
}
```

## 列表

&nbsp;&nbsp;大多数 UI 展示其实就是展示一组列表，而列表每个 item 的结构相同但是数据不一样。为了管理列表中每个 item 的状态，我们实现了一个 Map 结构来保存特定 item 的状态，这就是 state 的实现来源。

```js
function UserList(users, likesPerUser, updateUserLikes) {
  return users.map(user =>
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

注意: 上面 FancyNameBox 函数在每次调用时需要传递多个不同参数，这种实现破坏了 memorization 的能力，因为 memorization 一次只能记住一个返回值，显然上面的调用导致 memorization 功能失效

## 连续性

&nbsp;&nbsp;不幸的是，由于 UI 可以由种类繁多且量大的列表构成，因此让开发者显式维护这些列表状态将变得很繁琐。我们可以将这些繁琐的过程通过函数延迟执行手段从关键业务逻辑中移除。而函数延迟执行手段，包括使用 currying 技术
(https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind) in JavaScript). 因此我们将 state 状态通过外部核心函数传入，从而避免繁琐过程。这就是 props 的实现来源，注意 props 并不会将减轻 UI 繁琐的状态管理过程，而是将这些状态管理过程移除在外管理

```js
function FancyUserList(users) {
  return FancyBox(UserList.bind(null, users));
}

const box = FancyUserList(data.users);
const resolvedChildren = box.children(likesPerUser, updateUserLikes);
const resolvedBox = {
  ...box,
  children: resolvedChildren
};
```

## State Map

&nbsp;&nbsp;从之前的描述可知，一旦开发中发现一些重复模式则可以使用组合形式来复用模式。我们可以将提取和传递 state 的逻辑下沉到可复用的底层函数

```js
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

&nbsp;&nbsp;一旦需要记忆列表中多个 items，则 memorization 实现将变得非常困难，因为需要平衡内存占用率和内存使用频率而设计出复杂的缓存算法；幸运的是，UI 的变化很少出现跨层级的变动，即 UI 树中相同位置的 UI 节点几乎稳定。tree 结构是实现 memorization 的有效策略，我们使用相同的策略来管理 state 的状态，这即是 VDOM 的实现来源

```js
function memoize(fn) {
  return function(arg, memoizationCache) {
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
    children.map(child =>
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

## Algebraic Effects

&nbsp;&nbsp;这部分实现被认为是 PITA 概念的一种，即在多层结构中传递变化，最希望实现的方案是直接在变化相关的节点传递而不是逐层下传。这就是 React 实现中 context 的来源。

&nbsp;&nbsp;有时候数据依赖并不会恰当好处地遵循抽象树，比如 UI 布局算法，当前节点需要知道子节点的数目才能确定这些节点的位置信息。而下面例子使用了一种称为[Algebraic Effects](http://math.andrej.com/eff/) 的 [proposed for ECMAScript](https://esdiscuss.org/topic/one-shot-delimited-continuations-with-effect-handlers). 这是一种函数式编程的数学思维，可以避免被单子所暴露的中间仪式

```js
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
