---
title: react-redux-source-code-read
description: "react-redux源码试探阅读"
pubDatetime: 2018-07-03 17:36:15
author: codinglobster
ogImage: "reactjs.png"
tags:
  - react
  - redux
---

# Redux-react 源码解析

react 提供了 view 层的依据数据刷新，redux 提供了可追溯的数据流体系，但是如何将两者很好的整合起来呢？

> react-redux，通过 connect 来连接组件和 store

## 订阅流程图

![react-redux](./react-redux-source-code-read/react-redux-process.png)
修改浏览器缩放率可以查看大图。

## 从 Provider 开始

Provider 组件十分简单，接受参数 store，并将 store 作为上下文内容进行广播。

```js
Provider.propTypes = {
  store: storeShape.isRequired,
  children: PropTypes.element.isRequired,
};
Provider.childContextTypes = {
  [storeKey]: storeShape.isRequired,
  [subscriptionKey]: subscriptionShape,
};
```

可以看到 provider 必须是有子组件的，不然它将没有任何意义。
同时它会广播 subscription，但是值为 Null,是为了区分 subcription 来源做的 hack 处理。

## Connet 方法

connect

```js
connect(
    mapStateToProps
    mapDispatchToProps
    mergeProps
    {
        ======== 默认参数 ========
        pure = true,
        areStatesEqual = strictEqual,
        areOwnPropsEqual = shallowEqual,
        areStatePropsEqual = shallowEqual,
        areMergedPropsEqual = shallowEqual,
        =========================
        ...extraOptions // 可以覆盖connect的参数，也可以覆盖connectHOC的参数
    }

    return connectAdvance(props)

)

```

connect 方法接收了三个主要函数用来描述组件与 store 之间的数据依赖，并进行了验证处理，保证三个传入参数符合要求。
connect 方法可以说是 connectAdvance 的简单配置版，默认提供了大部分 selectorFactory 需要的参数。

### mapStateToProps

描述了组件与 state 之间的依赖，即需要哪些 state 中的属性
接受两个参数，state, ownProps

```js
const mapStateToProps = state => {
  return {
    todos: getVisibleTodos(state.todos, state.visibilityFilter),
  };
};
```

### mapDispatchToProps

接受两个参数（dispath,和 ownProps）
为组件的 props 上添加预设好的 dispath-action

```js
const mapDispatchToProps = (dispatch, ownProps) => {
  return {
    onClick: () => {
      dispatch({
        type: "SET_VISIBILITY_FILTER",
        filter: ownProps.filter,
      });
    },
  };
};
```

如果传入了这个参数，那么我们就可以在组件中使用`this.props.onClick()`就可以派发一个已经组装的  好的 action

### mergeProps

接受三个参数（`stateProps, dispatchProps, ownProps`）
描述了计算后的 state,dispatchProps,和之前的 props 是如何组合起来的
默认的组合方式是浅拷贝三个数据，并组成一个新的对象，即`{...stateProps, ...dispatchProps, ...ownProps}`

### 比对等级

默认的 state 比较是采用严格对比，也就是 `a === b`,
而其它属性则使用了浅比较。
只比较了 OwnProperty 相等即可。

```js
const hasOwn = Object.prototype.hasOwnProperty;

function is(x, y) {
  if (x === y) {
    return x !== 0 || y !== 0 || 1 / x === 1 / y;
  } else {
    return x !== x && y !== y;
  }
}

export default function shallowEqual(objA, objB) {
  if (is(objA, objB)) return true;

  if (
    typeof objA !== "object" ||
    objA === null ||
    typeof objB !== "object" ||
    objB === null
  ) {
    return false;
  }

  const keysA = Object.keys(objA);
  const keysB = Object.keys(objB);

  if (keysA.length !== keysB.length) return false;

  for (let i = 0; i < keysA.length; i++) {
    if (!hasOwn.call(objB, keysA[i]) || !is(objA[keysA[i]], objB[keysA[i]])) {
      return false;
    }
  }

  return true;
}
```

## ConnectAdvance

```JS
connectAdvanced(
    selectorFactory,(connect提供)
  {
    ...options
  }
){
  retrun wrapWithConnect（WrappedComponent）{
    class connect extends React.Component {
      ...
      render(
        <wrapWithConnect {...props} />
      )
    }
    return connect
  }
}
```

ConnectAdvance 会根据传入参数返回 wrap 方法，这个方法接收一个 element 组件。最终再返回一个普通的 connect 组件用来管理订阅和更新。

## 订阅的实现：subscribution

在返回的 connect 组件中，在初始化的过程中会初始化两个对象，updater 和 subsciption

```js
constructor(props, context) {

  ...

  this.state = {
    updater: this.createUpdater() // 初始化updater，下面会讲
  }
  this.initSubscription() // 初始化订阅
}
```

下面来看一看 subscription 的实现

```js
initSubscription() {
  if (!shouldHandleStateChanges) return // 没有mapsToProps则代表不需要监听，直接取消订阅，updater也就失去了意义
  const parentSub = (this.propsMode ? this.props : this.context)[subscriptionKey] // 如果父级已经有订阅，则直接订阅父级，从而实现父级刷新再刷新子级的逻辑
  this.subscription = new Subscription(this.store, parentSub, this.onStateChange.bind(this))
  // hack处理，因为订阅派发的过程可能会出现组件卸载的情况，所以，卸载前将订阅清空是必须的
  this.notifyNestedSubs = this.subscription.notifyNestedSubs.bind(this.subscription)
}
```

订阅类长什么样？

```js
export default class Subscription {
  constructor(store, parentSub, onStateChange) {
    this.store = store;
    this.parentSub = parentSub;
    this.onStateChange = onStateChange;
    this.unsubscribe = null;
    this.listeners = nullListeners;
  }

  addNestedSub(listener) {
    this.trySubscribe();
    return this.listeners.subscribe(listener);
  }

  notifyNestedSubs() {
    this.listeners.notify();
  }

  isSubscribed() {
    return Boolean(this.unsubscribe);
  }

  trySubscribe() {
    if (!this.unsubscribe) {
      this.unsubscribe = this.parentSub
        ? this.parentSub.addNestedSub(this.onStateChange)
        : this.store.subscribe(this.onStateChange);

      this.listeners = createListenerCollection();
    }
  }

  tryUnsubscribe() {
    if (this.unsubscribe) {
      this.unsubscribe();
      this.unsubscribe = null;
      this.listeners.clear();
      this.listeners = nullListeners;
    }
  }
}
```

可以看到订阅类实现了类似 redux 的订阅体系，但是要更复杂，订阅不仅要去`store`订阅`onStateChange`函数，还要管理可能存在的子组件订阅，可以看到实现的核心是`trySubscribe`，分两种情况去管理订阅。如果有`parentSub`，则将`onStateChange`注册到父级的 listeners 中，没有则直接在`store`中 subscribe。

那么问题又来了，什么时候开始订阅？以及 `onStateChange`函数到底干了什么？

```js
 componentDidMount() {
  if (!shouldHandleStateChanges) return
  this.subscription.trySubscribe()
  this.runUpdater()
 }

 onStateChange() {
   this.runUpdater(this.notifyNestedSubs)
 }
```

在组件加载完成后，组件开始了订阅，并且进行了一次手动更新。
而`onStateChange`则是  将子组件的订阅作为回调传入`runUpdater`

所以`runUpdater`又干了什么？

```js
runUpdater(callback = noop) {
  if (this.isUnmounted) {
    return
  }
  this.setState(prevState => prevState.updater(this.props, prevState), callback)
}
```

刷新的关键就在这里，通过 setState,我们将  通过`Updater`计算的结果传入，从而进行更新，并在更新完成之后再进行子组件的更新。

## 更新的实现：updater & SelectFactory

上一节我们看到，在组件初始化的时候，我们注册了 Updater，那么他是怎么实现的呢？

```js
createUpdater() {
  const sourceSelector = selectorFactory(this.store.dispatch, selectorFactoryOptions)
  return makeUpdater(sourceSelector, this.store)
}
// 这个是组件外部封装的方法
function makeUpdater(sourceSelector, store) {
  return function updater(props, prevState) {
    try {
      const nextProps = sourceSelector(store.getState(), props)
      if (nextProps !== prevState.props || prevState.error) {
        return {
          shouldComponentUpdate: true,
          props: nextProps,
          error: null,
        }
      }
      return {
        shouldComponentUpdate: false,
      }
    } catch (error) {
      return {
        shouldComponentUpdate: true,
        error,
      }
    }
  }
}
```

```js
export default function finalPropsSelectorFactory(
  dispatch,
  { initMapStateToProps, initMapDispatchToProps, initMergeProps, ...options }
) {
  const mapStateToProps = initMapStateToProps(dispatch, options);
  const mapDispatchToProps = initMapDispatchToProps(dispatch, options);
  const mergeProps = initMergeProps(dispatch, options);

  if (process.env.NODE_ENV !== "production") {
    verifySubselectors(
      mapStateToProps,
      mapDispatchToProps,
      mergeProps,
      options.displayName
    );
  }

  const selectorFactory = options.pure
    ? pureFinalPropsSelectorFactory
    : impureFinalPropsSelectorFactory;

  return selectorFactory(
    mapStateToProps,
    mapDispatchToProps,
    mergeProps,
    dispatch,
    options
  );
}
```

可以看到传入的主要参数是 `dispatch,initMapStateToProps,initMapDispatchToProps,initMergeProps`和`pure,connect`组件中传入的三个匹配的方法需要传入`dispatch`和`option`才能使用

selectFactory 最后返回的也是

```js
return function pureFinalPropsSelector(nextState, nextOwnProps) {
  return hasRunAtLeastOnce
    ? handleSubsequentCalls(nextState, nextOwnProps)
    : handleFirstCall(nextState, nextOwnProps);
};
```

接受 state,和 props，并与自己缓存的 ownProps, state 比对后返回整合后的 nextProps
结合`makeUpdater`所以上面接受的 state 是 store 的 state, props 是 updater 传入的,也就是当前 connect 组件的 props

```js
export function pureFinalPropsSelectorFactory(
  mapStateToProps,
  mapDispatchToProps,
  mergeProps,
  dispatch,
  { areStatesEqual, areOwnPropsEqual, areStatePropsEqual }
) {
  let hasRunAtLeastOnce = false;
  let state;
  let ownProps;
  let stateProps;
  let dispatchProps;
  let mergedProps;

  function handleFirstCall(firstState, firstOwnProps) {
    state = firstState;
    ownProps = firstOwnProps;
    stateProps = mapStateToProps(state, ownProps);
    dispatchProps = mapDispatchToProps(dispatch, ownProps);
    mergedProps = mergeProps(stateProps, dispatchProps, ownProps);
    hasRunAtLeastOnce = true;
    return mergedProps;
  }

  function handleNewPropsAndNewState() {
    stateProps = mapStateToProps(state, ownProps);

    if (mapDispatchToProps.dependsOnOwnProps)
      dispatchProps = mapDispatchToProps(dispatch, ownProps);

    mergedProps = mergeProps(stateProps, dispatchProps, ownProps);
    return mergedProps;
  }

  function handleNewProps() {
    if (mapStateToProps.dependsOnOwnProps)
      stateProps = mapStateToProps(state, ownProps);

    if (mapDispatchToProps.dependsOnOwnProps)
      dispatchProps = mapDispatchToProps(dispatch, ownProps);

    mergedProps = mergeProps(stateProps, dispatchProps, ownProps);
    return mergedProps;
  }

  function handleNewState() {
    const nextStateProps = mapStateToProps(state, ownProps);
    const statePropsChanged = !areStatePropsEqual(nextStateProps, stateProps);
    stateProps = nextStateProps;

    if (statePropsChanged)
      mergedProps = mergeProps(stateProps, dispatchProps, ownProps);

    return mergedProps;
  }

  function handleSubsequentCalls(nextState, nextOwnProps) {
    const propsChanged = !areOwnPropsEqual(nextOwnProps, ownProps);
    const stateChanged = !areStatesEqual(nextState, state);
    state = nextState;
    ownProps = nextOwnProps;

    if (propsChanged && stateChanged) return handleNewPropsAndNewState();
    if (propsChanged) return handleNewProps();
    if (stateChanged) return handleNewState();
    return mergedProps;
  }

  return function pureFinalPropsSelector(nextState, nextOwnProps) {
    return hasRunAtLeastOnce
      ? handleSubsequentCalls(nextState, nextOwnProps)
      : handleFirstCall(nextState, nextOwnProps);
  };
}
```

通常我们使用的就是 pureFinalPropsSelectorFactory，通过缓存之前的 state 和 props 和传入的 state，props 进行指定的比对，从而判断是否需要更新，以及怎么样  更新。

## 被包裹组件的更新

updater 运行之后，setState 只是改变 connect 组件的 props，被包裹的组件时怎么更新的呢？

```js
addExtraProps(props) {
  if (!withRef && !renderCountProp && !(this.propsMode && this.subscription)) return props
  const withExtras = { ...props }
  if (withRef) withExtras.ref = this.setWrappedInstance
  if (renderCountProp) withExtras[renderCountProp] = this.renderCount++
  if (this.propsMode && this.subscription) withExtras[subscriptionKey] = this.subscription
  return withExtras
}

render() {
  if (this.state.error) {
    throw this.state.error
  } else {
    return createElement(WrappedComponent, this.addExtraProps(this.state.props))
  }
}
```

可以看到，render 函数是将被包裹的组件注入了所有的 Props 后返回的，所以只要 connect 的 props 发生了变化，被包裹的组件就会相应的刷新。

## 静态属性相关

为了不影响到子组件的内部实现，connect 组件将子组件的静态属性进行了拷贝，防止 propType，contextType 这些属性的失效，实现的途径是使用第三方类库`hoist-non-react-statics`

```js
Connect.WrappedComponent = WrappedComponent; // 最原始的被包裹组件
Connect.displayName = displayName; // 组件的现实名称
Connect.childContextTypes = childContextTypes; // 子级上线文限制
Connect.contextTypes = contextTypes; // 自身上下文限制
Connect.propTypes = contextTypes; // 自身属性限制，会和子组件属性进行合并
Connect.getDerivedStateFromProps = getDerivedStateFromProps; // react 新增生命周期，返回更新后的state,用来更新组件

return hoistStatics(Connect, WrappedComponent);
```

## 包裹组件如何获取

在`connectAdvance`中，`withRef` 为 true 时，connect 组件会将 wrapComponent 的 ref 属性覆盖，将 ref 的回调替换为

```JS
setWrappedInstance(ref) {
    this.wrappedInstance = ref
}
```

所以当引用当前 connect 组建时可以通过，获取被包裹组件的实例

```js
getWrappedInstance() {
    invariant(withRef,
        `To access the wrapped instance, you need to specify ` +
        `{ withRef: true } in the options argument of the ${methodName}() call.`
    )
    return this.wrappedInstance
}
```

通过当前 api 也可以获取，但是这是最原始的组件，并不是实例
`Connect.WrappedComponent = WrappedComponent`

## shouldHandleStateChange 和 pure 的区别

`shouldHandleStateChange`决定了是否需要订阅
`pure`则是决定了 selector 更新的方式,pure 为 false 时，组件将会跳过比较阶段，直接将所有属性更新。

## propsMode

connect 组件没有强制规定只能使用 provider 提供的 store,即你可以直接将其它 store 直接通过 Props 传递给 connect 组件，但这就造成了订阅可能出现混乱的问题，
所以需要对 store 来自 props 的组件进行区分。

![react-redux](./props-mode.png)

### 初始化的区别

```js
initSubscription() {
  if (!shouldHandleStateChanges) return

  // parentSub's source should match where store came from: props vs. context. A component
  // connected to the store via props shouldn't use subscription from context, or vice versa.
  const parentSub = (this.propsMode ? this.props : this.context)[subscriptionKey]
  this.subscription = new Subscription(this.store, parentSub, this.onStateChange.bind(this))

  this.notifyNestedSubs = this.subscription.notifyNestedSubs.bind(this.subscription)
}
```

源码也进行了解释，订阅需要和它的来源相对应，即 store 从 props 来，他的订阅也应该是 prop 上的，反之亦然

### 上下文的区别

```js
getChildContext() {
  // If this component received store from props, its subscription should be transparent
  // to any descendants receiving store+subscription from context; it passes along
  // subscription passed to it. Otherwise, it shadows the parent subscription, which allows
  // Connect to control ordering of notifications to flow top-down.
  const subscription = this.propsMode ? null : this.subscription
  return { [subscriptionKey]: subscription || this.context[subscriptionKey] }
}
```

connect 组件会将 subsciption 在上下文上传递，但如果是通过 props 传入 store 的组件是不需要向下传递的，他的订阅应该是透明的。

### wrapElementProps 的区别

```js
addExtraProps(props) {
  if (!withRef && !renderCountProp && !(this.propsMode && this.subscription)) return props
  // make a shallow copy so that fields added don't leak to the original selector.
  // this is especially important for 'ref' since that's a reference back to the component
  // instance. a singleton memoized selector would then be holding a reference to the
  // instance, preventing the instance from being garbage collected, and that would be bad
  const withExtras = { ...props }
  if (withRef) withExtras.ref = this.setWrappedInstance
  if (renderCountProp) withExtras[renderCountProp] = this.renderCount++
  if (this.propsMode && this.subscription) withExtras[subscriptionKey] = this.subscription
  return withExtras
}
```

可以看到，propsMode 的订阅会通过 props 的模式传递给被包裹的组件，如果对源码熟悉的话，可以进行一系列的操作。

## 后记

`redux`和`react-redux`组合起来，数据管理，以及组件的  数据订阅已经相当完善了，但是还有一个问题没有解决，如何更好的处理异步请求 ?
`redux-saga`是目前最受欢迎的异步处理框架,因为它没有破坏 dispatch 的封装性，并且能够以同步代码的形式书写异步代码，下一集我们再来讨论它吧。
