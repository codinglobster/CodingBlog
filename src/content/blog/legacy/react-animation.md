---
title: react-animation-source-code-read
description: "react-animation源码试探阅读"
pubDatetime: 2018-11-19 14:21:13
author: codinglobster
tags:
  - react
---

# React 动画之社区实现

react 动画实现由`Transition`、`CSSTransition`、`TransitionGroup`组成，三个组件分别实现了动画状态管理，动画 css 管理，和列表动画元素状态管理。

## Transition

`transition`组件是实现动画状态监管的核心组件，它本身并不实现任何动画逻辑，只是保存了当前组件的动画状态，和状态切换时的钩子函数调用。

```js
export const UNMOUNTED = "unmounted"; // 未渲染
export const EXITED = "exited"; // 退场的
export const ENTERING = "entering"; // 进入中
export const ENTERED = "entered"; // 进入的
export const EXITING = "exiting"; // 退出的
```

以上是源码中定义的组件状态。

### constuctor

```js

  constructor(props, context) {
    super(props, context)

    let parentGroup = context.transitionGroup
    // In the context of a TransitionGroup all enters are really appears
    let appear =
      parentGroup && !parentGroup.isMounting ? props.enter : props.appear

    let initialStatus

    this.appearStatus = null

    if (props.in) {
      if (appear) {
        initialStatus = EXITED
        this.appearStatus = ENTERING
      } else {
        initialStatus = ENTERED
      }
    } else {
      if (props.unmountOnExit || props.mountOnEnter) {
        initialStatus = UNMOUNTED
      } else {
        initialStatus = EXITED
      }
    }

    this.state = { status: initialStatus }

    this.nextCallback = null
  }
```

构造器函数中，首先根据传入的 Props 来确认组件的初始状态。

### getDerivedStateFromProps

```js
 static getDerivedStateFromProps({ in: nextIn }, prevState) {
    if (nextIn && prevState.status === UNMOUNTED) {
      return { status: EXITED }
    }
    return null
  }
```

react 在最近的 16.4 中将该生命周期修改为`static`，并将函数的调用机制修改为贯穿创建和更新。具体可以查看官方提供的示意图。
[react 生命周期](http://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/)
这段代码的的作用是如果下一次组件的状态是`准备进入`但是之前的状态是未渲染的，这直接将状态替换为`退出的`

首先确认下`UNMOUTED`的状态是如何产生的，只有在组件上添加了`unmountOnExit`或者`mountOnEnter`，且初始化状态`in`为`false`时，会在初始化阶段定义为该状态。

### render

```js
render() {
    const status = this.state.status
    if (status === UNMOUNTED) {
      return null
    }

    const { children, ...childProps } = this.props
    // filter props for Transtition
    delete childProps.in
    delete childProps.mountOnEnter
    delete childProps.unmountOnExit
    delete childProps.appear
    delete childProps.enter
    delete childProps.exit
    delete childProps.timeout
    delete childProps.addEndListener
    delete childProps.onEnter
    delete childProps.onEntering
    delete childProps.onEntered
    delete childProps.onExit
    delete childProps.onExiting
    delete childProps.onExited

    if (typeof children === 'function') {
      return children(status, childProps)
    }

    const child = React.Children.only(children)
    return React.cloneElement(child, childProps)
  }
```

接上面的，当状态为`UNMOUNTED`时，是不会渲染内容的，所以如果要进入是需要从`UNMOUNTED => EXITED => ENTERED`,当然，如果定义了`unmountOnExit`，当`in`为`false`的时候，组件的状态会退回为`UNMOUNTED`。

render 函数支持两种渲染方式，函数和 react element,所以需要将内部使用的 prop 删除掉再传递给子组件。

### didMount & didUpdate

组件在渲染完成之后，有了初始的状态，为了动画开始，需要对组件的状态进行修改。

```js
updateStatus(mounting = false, nextStatus) {
    if (nextStatus !== null) {
      // nextStatus will always be ENTERING or EXITING.
      this.cancelNextCallback()
      const node = ReactDOM.findDOMNode(this)

      if (nextStatus === ENTERING) {
        this.performEnter(node, mounting)
      } else {
        this.performExit(node)
      }
    } else if (this.props.unmountOnExit && this.state.status === EXITED) {
      this.setState({ status: UNMOUNTED })
    }
  }
```

渲染完成和更新的钩子会根据状态分别执行`performEnter`和`performExit`

```js
performEnter(node, mounting) {
    const { enter } = this.props
    const appearing = this.context.transitionGroup
      ? this.context.transitionGroup.isMounting
      : mounting

    const timeouts = this.getTimeouts()

    // no enter animation skip right to ENTERED
    // if we are mounting and running this it means appear _must_ be set
    if (!mounting && !enter) {
      this.safeSetState({ status: ENTERED }, () => {
        this.props.onEntered(node)
      })
      return
    }

    this.props.onEnter(node, appearing)

    this.safeSetState({ status: ENTERING }, () => {
      this.props.onEntering(node, appearing)

      // FIXME: appear timeout?
      this.onTransitionEnd(node, timeouts.enter, () => {
        this.safeSetState({ status: ENTERED }, () => {
          this.props.onEntered(node, appearing)
        })
      })
    })
  }
```

可以看到，通过初始状态，逐步调用`onEnter,onEntering,onEntered`并且更替状态
`ENTERING => ENTERED`。
而我们需要的做的就是根据 Transition 组件返回的状态来实现代码逻辑。

## CSSTransition

CSSTransition 的实现就比较简单了，可以直接看 render 就可以了

```js
render() {
    const props = { ...this.props };

    delete props.classNames;

    return (
      <Transition
        {...props}
        onEnter={this.onEnter}
        onEntered={this.onEntered}
        onEntering={this.onEntering}
        onExit={this.onExit}
        onExiting={this.onExiting}
        onExited={this.onExited}
      />
    );
  }
```

实际上就是对 Trsition 的一个扩展，作用就是在状态变更的时候加上相应的 class 名称，如果提前在环境中定义了相应的 css 定义，则直接可以实现效果，减少了代码。
用过 vue transition 组件的同学应该很熟悉。

## TransitionGroup

TransitionGroup 相当于是在 CSSTransition 的基础上再进一步的封装。
为包裹的子组件提供相应的状态。

```js
static getDerivedStateFromProps(
    nextProps,
    { children: prevChildMapping, handleExited, firstRender }
  ) {
    return {
      children: firstRender
        ? getInitialChildMapping(nextProps, handleExited)
        : getNextChildMapping(nextProps, prevChildMapping, handleExited),
      firstRender: false,
    }
  }
```

可以看到，组件并不是直接进行渲染 children,而是进行了一层封装。

```js
export function getInitialChildMapping(props, onExited) {
  return getChildMapping(props.children, child => {
    return cloneElement(child, {
      onExited: onExited.bind(null, child),
      in: true,
      appear: getProp(child, "appear", props),
      enter: getProp(child, "enter", props),
      exit: getProp(child, "exit", props),
    });
  });
}

export function getChildMapping(children, mapFn) {
  let mapper = child => (mapFn && isValidElement(child) ? mapFn(child) : child);

  let result = Object.create(null);
  if (children)
    Children.map(children, c => c).forEach(child => {
      // run the map function here instead so that the key is the computed one
      result[child.key] = mapper(child);
    });
  return result;
}
```

可以看到，封装之后的子组件已经不是之前的了，而且经过克隆的组件会添加默认属性，而且如果之前定义过，则会覆盖。

```js
 render() {
    const { component: Component, childFactory, ...props } = this.props
    const children = values(this.state.children).map(childFactory)

    delete props.appear
    delete props.enter
    delete props.exit

    if (Component === null) {
      return children
    }
    return <Component {...props}>{children}</Component>
  }
```

render 和之前的实现很相似。

以上是组件进入时候的初始化，但是当组件中子组件数量发生变化的时候又是如何处理的呢？

`getNextChildMapping`

```js
export function getNextChildMapping(nextProps, prevChildMapping, onExited) {
  let nextChildMapping = getChildMapping(nextProps.children);
  let children = mergeChildMappings(prevChildMapping, nextChildMapping);

  Object.keys(children).forEach(key => {
    let child = children[key];

    if (!isValidElement(child)) return;

    const hasPrev = key in prevChildMapping;
    const hasNext = key in nextChildMapping;

    const prevChild = prevChildMapping[key];
    const isLeaving = isValidElement(prevChild) && !prevChild.props.in;

    // item is new (entering)
    if (hasNext && (!hasPrev || isLeaving)) {
      // console.log('entering', key)
      children[key] = cloneElement(child, {
        onExited: onExited.bind(null, child),
        in: true,
        exit: getProp(child, "exit", nextProps),
        enter: getProp(child, "enter", nextProps),
      });
    } else if (!hasNext && hasPrev && !isLeaving) {
      // item is old (exiting)
      // console.log('leaving', key)
      children[key] = cloneElement(child, { in: false });
    } else if (hasNext && hasPrev && isValidElement(prevChild)) {
      // item hasn't changed transition states
      // copy over the last transition props;
      // console.log('unchanged', key)
      children[key] = cloneElement(child, {
        onExited: onExited.bind(null, child),
        in: prevChild.props.in,
        exit: getProp(child, "exit", nextProps),
        enter: getProp(child, "enter", nextProps),
      });
    }
  });

  return children;
}
```

第一步，clone 新的 children,
第二步，合并之前的 children,这里的合并逻辑是将子组件已相对合理的顺序进行整合。然后根据状态的不同，分别进入和离开。
第三部，根据状态的不同，clone 新的子对象，并添加相应的属性，依旧会覆盖子组件的定义。
第四部，render

以上就是社区实现的动画组件，了解过其实现原理后，再写一些简单的过渡动画，那就再棒不过了。
