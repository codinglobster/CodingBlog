---
# layout: post
title: redux-source-code-read
description: "redux源码试探阅读"
pubDatetime: 2018-07-01 12:30:11
author: codinglobster
ogImage: "redux-bg.png"
tags:
  - react
  - redux
---

# Redux 源码解析

## redux 的作用

![redux](./redux-example-css-tricks-opt.jpg)

没有统一的数据处理，数据在组件中不断传递，很多时候无法准确的定位程序在哪里出错了。
而使用 redux 将需要统一处理的数据交由 store 管理，state 只能通过 dispatch 来修改，并且修改过后统一通知，这样数据的变化变得可追溯，数据变化后的变化也可预测。

## 为什么不使用 context

```js
const PropTypes = require('prop-types');

class MediaQuery extends React.Component {
  ...
  // 如果在shouldComponentUpdate返回为空则改方法不会触发，子组件context不会刷新
  getChildContext() {
    return {type: this.state.type};
  }
  ...
}

MediaQuery.childContextTypes = {
  type: PropTypes.string
};
```

使用 context 的限制在于声明周期，如果一个数据中间组件负责转发 context，但是 shouldComponentUpdate 声明周期中返回 false 就会导致接受 context 的子组件无法刷新。

## 原作者对于 redux 的解释

![redux](./concept.jpeg)

## 三个概念

1. dispatch 可追溯数据流模式（单向数据流）
2. middleware 中间件模式（为 dispatch 增加更多的可能性）
3. snapshot 快照模式（管理订阅）

## 目录结构

```
├── applyMiddleware.js
├── bindActionCreators.js
├── combineReducers.js
├── compose.js
├── createStore.js
├── index.js
└── utils
    ├── actionTypes.js
    ├── isPlainObject.js
    └── warning.js
```

## 工具类介绍

- **actionTypes** (redux 内部使用变量，防止局部出现重复名称)
- **warning** (如果环境存在 console.error 则将错误打印在控制台，如果没有，则将异常抛出，配合 break all on exceptions 可以在抛出错误行暂停)
- **isPlainObject** (判断一个对象是否是简单对象，简单对象的定义就是他的原型链只有一层)

---

## 从 createStore 开始

```js
export default function createStore(reducer, preloadedState, enhancer) {

  let currentReducer = reducer // 输入一个state，返回一个新的state
  let currentState = preloadedState // store的state
  let currentListeners = [] // store订阅的监听器
  let nextListeners = currentListeners // 用于注册监听器的容器
  let isDispatching = false // 判断当前是否存在dispatch操作

  function ensureCanMutateNextListeners(){}
  function getState() {}
  function subscribe(listener) {}
  function dispatch(action) {}
  function replaceReducer(nextReducer) {}
  function observable() {}

  dispatch({ type: ActionTypes.INIT })

  return {
    dispatch,
    subscribe,
    getState,
    replaceReducer,
    [$$observable]: observable
  }

```

**store**中的变量共五个，**currentReducer**负责 dispatch 后重新计算 state，**preloadedState**负责默认初始的 state 的值，**currentListeners**和**nextListeners**负责管理订阅的函数，**isDispatching**则是起到判断边界的作用，dispatch 使用也是有一定的限制。

核心代码应该算是 dispatch 了吧

```js
function dispatch(action) {
  // action必须是简单对象
  if (!isPlainObject(action)) {
    throw new Error(
      "Actions must be plain objects. " +
        "Use custom middleware for async actions."
    );
  }
  // action 必须有type
  if (typeof action.type === "undefined") {
    throw new Error(
      'Actions may not have an undefined "type" property. ' +
        "Have you misspelled a constant?"
    );
  }
  // 同一时间，dispatch不能同时进行
  if (isDispatching) {
    throw new Error("Reducers may not dispatch actions.");
  }

  try {
    isDispatching = true; // 标记开始，其它dispatch不能在进行派发
    currentState = currentReducer(currentState, action); // 通过reducer重新计算新的state
  } finally {
    isDispatching = false; // 派发完毕
  }
  // 计算完成后，将所有注册的listeners运行一遍
  const listeners = (currentListeners = nextListeners);
  for (let i = 0; i < listeners.length; i++) {
    const listener = listeners[i];
    listener();
  }

  return action;
}
```

结合作者的示意图，redux 的流程就算是相当的清楚了吧。

`dispatch action >> state change >> view change`

## combineReducer

函数的作用是将多个 reducer 组合起来，并为每个 reducer 分配对应 key 值的 namespace，更重要的是通过三次判断，判断传入的 reducer 是否符合规范

1. **_assertReducerShape_**

   - reducerstate 必须有默认值，并且不能使用内部的 actionTypes，

2. **_getUnexpectedStateShapeWarningMessage_**（组装好的 re）

   - 对当前 dispatch 进行判断，是第一次 dispatch
   - 对 redecuer 对象的长度进行判断
   - state 必须是简单对象
   - state 的 key 和 reducer 的 key 进行比较，找出无效的 Key
     这个操作对 replace 操作是无效的，因为替换 reducer 就可能会导致 key 值得变化

3. **_getUndefinedStateErrorMessage_**
   - 对传入的 state 取对应的值，并经过 reducer 处理后返回 undefined 的，进行报错处理
   ```
   `Given ${actionDescription}, reducer "${key}" returned undefined. ` +
   `To ignore an action, you must explicitly return the previous state. ` +
   `If you want this reducer to hold no value, you can return null instead of undefined.`
   ```

## actionCreater

作用就是返回一个 action

```js
export function addTodo(text) {
  return { type: ADD_TODO, text };
}
```

redux-**bindActionCreators**

```js
function bindActionCreator(actionCreator, dispatch) {
  return function () {
    return dispatch(actionCreator.apply(this, arguments));
  };
}
```

berfore

```js
store.dispath(addTodo(text));
```

after

```js
const newActionCreater = bindActionCreators(addTodo, dispatch);
newActionCreater(text); // action 已经被派发
```

## applyMiddleware

```js
export default function applyMiddleware(...middlewares) {
  return createStore =>
    (...args) => {
      const store = createStore(...args);
      let dispatch = () => {
        throw new Error(
          `Dispatching while constructing your middleware is not allowed. ` +
            `Other middleware would not be applied to this dispatch.`
        );
      };

      const middlewareAPI = {
        getState: store.getState,
        dispatch: (...args) => dispatch(...args),
      };
      const chain = middlewares.map(middleware => middleware(middlewareAPI));
      dispatch = compose(...chain)(store.dispatch);

      return {
        ...store,
        dispatch,
      };
    };
}
```

如果 createStore 参数中传入了 enhancer 参数，则初始化的过程会发生在 applyMiddleware 中，dispath 方法会经过改造，成为中间件形式，最原始的 dispatch 会被包裹在最中心，中间件们可以在 dispatch 前和 dispatch 后进行一系列操作，如 redux-log,redux-thunk,redux-saga

> 问题：为什么 dispath 要写成闭包的形式？

因为为了避免在注册初始有人 dispatch，所以 dispatch 暂时会成为一个报告异常函数，而如果直接将 dispath 赋值给`middlewareAPI`，dispatch 的引用将不会在产生变化，也就是即使下面 dispatch 已经被替换为经过 compose 的  函数，dispatch 仍然是抛出异常的函数。
闭包就可以避免这个问题，任何时间，dispatch 发生变化，middlewareApi 中的 dispatch 也会得到更新。

## compose

```js
export default function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg;
  }

  if (funcs.length === 1) {
    return funcs[0];
  }

  return funcs.reduce(
    (a, b) =>
      (...args) =>
        a(b(...args))
  );
}
```

洋葱圈模型的基础，传入的方法会使用 reduce 函数将数组函数转换为嵌套调用的形式

`compose(a, b, c) 变为 (...args) =>  a(b(c(...args)))`

## index

```js
function isCrushed() {}

if (
  process.env.NODE_ENV !== "production" &&
  typeof isCrushed.name === "string" &&
  isCrushed.name !== "isCrushed"
) {
  warning(
    'You are currently using minified code outside of NODE_ENV === "production". ' +
      "This means that you are running a slower development build of Redux. " +
      "You can use loose-envify (https://github.com/zertosh/loose-envify) for browserify " +
      "or setting mode to production in webpack (https://webpack.js.org/concepts/mode/) " +
      "to ensure you have the correct code for your production build."
  );
}
```

用于判断文件是否被压缩的方法

## 结语

redux 提供了一个高完成度的外部数据管理体系，我们可以很容易的在 react 之外管理数据，但是如何将 redux 的数据和 react 组件相结合呢？

官方给出了`react-redux`的解决方案，下一篇我们再来讨论吧。
