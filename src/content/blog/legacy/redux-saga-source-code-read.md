---
title: redux-saga-source-code-read
description: "redux-saga源码试探阅读"
pubDatetime: 2018-07-04 17:24:39
author: codinglobster
ogImage: "redux-saga.png"
tags:
  - react
  - redux
---

## 前言

在使用`dva`框架的过程中，我一直很好奇`effect`是如何实现将异步代码写成了同步代码的形式，直到看到了`dva知识图谱`才了解到，异步处理这块是交由`redux-saga`来处理的，所以就有了这篇试探阅读代码，来看看到底自己能不能看懂这优雅的异步处理源码。

## 关于生成器函数(generator function)

`redux-saga`的核心就是生成器函数,所以首先来弄清楚生成器函数的基本用法吧。

```js
function* generator(i) {
  yield i;
  yield i + 10;
}

var gen = generator(10);

console.log(gen.next().value);
// expected output: 10

console.log(gen.next().value);
// expected output: 20
```

生成器函数和普通函数类似，只是在`function`后跟了一个`*`号，但是不能使用`new`操作符来当构造函数使用。
看看 mdn 怎么介绍这个函数

> 调用一个生成器函数并不会马上执行它里面的语句，而是返回一个这个生成器的 迭代器 （iterator ）对象。当这个迭代器的 next() 方法被首次（后续）调用时，其内的语句会执行到第一个（后续）出现 yield 的位置为止，yield 后紧跟迭代器要返回的值。或者如果用的是 yield\*（多了个星号），则表示将执行权移交给另一个生成器函数（当前生成器暂停执行）。

因为这种特殊的机制，非常适合分步处理，但`redux-saga`到底是怎样做到的呢？

## 基本流程

![redline](./redux-saga-process.png)

这是我绘制的 saga 的基本流程图，之后的解析也是参照这个流程来书写。

## 关于 effect

源码中暴露出来的 effects

```js
export {
  take,
  takeMaybe,
  put,
  putResolve,
  all,
  race,
  call,
  apply,
  cps,
  fork,
  spawn,
  join,
  cancel,
  select,
  actionChannel,
  cancelled,
  flush,
  getContext,
  setContext,
  delay,
} from "./internal/io";
```

初次接触 effect 的时候，天真的以为，saga 提供的函数会直接运行，并将结果返回，但某日查看官方文档，才发现，saga 是声明式 effect,当我们调用 effect 的时候，实际上是生成了一个描述当前行为的对象，如调用

```js
// 调用
yield call(Api.fetch, '/products')
// 返回结果
// Effect -> 调用 Api.fetch 函数并传递 `./products` 作为参数
{
CALL: {
  fn: Api.fetch,
  args: ['./products']
}
}
```

声明式调用的好处就是调用后返回结果一致， 方便测试的进行.

## 关于 middleware

作为一个`redux`的`middleware`,redux-saga 拥有获取 action 和 state 的能力，我们也知道 redux-saga 通过 take effect 来获取 action 的监听，所以 saga 是如何处理这些 take 请求并触发对应的 effect 的呢？

```js
import { createStore, applyMiddleware } from 'redux'
import createSagaMiddleware from 'redux-saga'
...
import reducer from './reducers'
import rootSaga from './sagas'
...
const sagaMiddleware = createSagaMiddleware({ sagaMonitor })
const store = createStore(reducer, applyMiddleware(sagaMiddleware))
sagaMiddleware.run(rootSaga)
```

这是最基础的初始化过程，我们就先从`createSagaMiddleware`开始吧。

```js
export default function sagaMiddlewareFactory({ context = {}, ...options } = {}) {
  const { sagaMonitor, logger, onError, effectMiddlewares } = options
  ...
  function sagaMiddleware({ getState, dispatch }) {
    const channel = stdChannel()
    channel.put = (options.emitter || identity)(channel.put)

    sagaMiddleware.run = runSaga.bind(null, {
      context,
      channel,
      dispatch,
      getState,
      sagaMonitor,
      logger,
      onError,
      effectMiddlewares,
    })

    return next => action => {
      if (sagaMonitor && sagaMonitor.actionDispatched) {
        sagaMonitor.actionDispatched(action)
      }
      const result = next(action) // hit reducers
      channel.put(action)
      return result
    }
  }

  sagaMiddleware.run = () => {
    throw new Error('Before running a Saga, you must mount the Saga middleware on the Store using applyMiddleware')
  }

  sagaMiddleware.setContext = props => {
    if (process.env.NODE_ENV === 'development') {
      check(props, is.object, createSetContextWarning('sagaMiddleware', props))
    }

    object.assign(context, props)
  }

  return sagaMiddleware
}
```

createSagaMiddleware 是作为 default 返回的，所以源码中它的名字是`sagaMiddlewareFactory`,它的作用就是初始化 middleware，并为 middleware 注册一些关键函数，如`run`和`setContext`;

我们也看到 saga 作为中间件的作用就是将`action` `put` 到`stdChannel`中去。

从 main 中我们了解到，当 saga 完成注册后需要`run(rootSaga)`，接下来我们就看看`runSaga`的实现。

```js
export function runSaga(options, saga, ...args) {
  if (process.env.NODE_ENV === 'development') {
    check(saga, is.func, NON_GENERATOR_ERR)
  }

  const iterator = saga(...args)

  if (process.env.NODE_ENV === 'development') {
    check(iterator, is.iterator, NON_GENERATOR_ERR)
  }

  const {
    channel = stdChannel(),
    dispatch,
    getState,
    context = {},
    sagaMonitor,
    logger,
    effectMiddlewares,
    onError,
  } = options

  const effectId = nextSagaId()

  ...

  ...

  ...

  const log = logger || _log
  const logError = err => {
    log('error', err)
    if (err && err.sagaStack) {
      log('error', err.sagaStack)
    }
  }

  const middleware = effectMiddlewares && compose(...effectMiddlewares)
  const finalizeRunEffect = runEffect => {
    if (is.func(middleware)) {
      return function finalRunEffect(effect, effectId, currCb) {
        const plainRunEffect = eff => runEffect(eff, effectId, currCb)
        return middleware(plainRunEffect)(effect)
      }
    } else {
      return runEffect
    }
  }

  const env = {
    stdChannel: channel,
    dispatch: wrapSagaDispatch(dispatch),
    getState,
    sagaMonitor,
    logError,
    onError,
    finalizeRunEffect,
  }

  const task = proc(env, iterator, context, effectId, getMetaInfo(saga), null)

  if (sagaMonitor) {
    sagaMonitor.effectResolved(effectId, task)
  }

  return task
}
```

`runSaga`实际上是对`proc`的一层调用封装，也就是在进行最后的处理前，再对`saga.run`进行进一步的配置，如添加对 runEffect 的封装，实现类似 redxu-middleware 的调用。添加报错处理，并为当前的 effect 添加 id,方便后面的处理。
最后，将所有的配置放入`env`，将`rootSaga`转变成为`iterator`对象。

# 解析 effects

一般来说，sagaMiddleware.run()注册的都是 watcher,也就是时刻都对 action 进行监听，并在相应的 action 派发时，执行对应的 effect。
所以说，saga 的第一步是注册 taker。
proc 函数:

```js
const taskContext = Object.create(parentContext);
const finalRunEffect = env.finalizeRunEffect(runEffect);

let crashedEffect = null;
const cancelledDueToErrorTasks = [];

next.cancel = noop;

const task = newTask(parentEffectId, meta, iterator, cont);
const mainTask = {
  meta,
  cancel: cancelMain,
  _isRunning: true,
  _isCancelled: false,
};

const taskQueue = forkQueue(
  mainTask,
  function onAbort() {
    cancelledDueToErrorTasks.push(...taskQueue.getTaskNames());
  },
  end
);

// kicks up the generator
next();

// then return the task descriptor to the caller
return task;
```

`saga`执行`proc`，并创建了`task`表示当前任务，`taskQueue`代表当前任务的任务队列，也就是说，任何一个 saga 是可以挂在其它`saga`让其在它的子线程运行，`fork-effect`的实现就是如此。
`next`会执行 saga,并返回 effect 的描述符，并将这个结果传递给`digestEffect`然后再经过判断来传递给`runEffect`,

```js
function runEffect(effect, effectId, currCb) {
  if (is.promise(effect)) {
    resolvePromise(effect, currCb);
  } else if (is.iterator(effect)) {
    resolveIterator(effect, effectId, meta, currCb);
  } else if (effect && effect[IO]) {
    const { type, payload } = effect;
    if (type === effectTypes.TAKE) runTakeEffect(payload, currCb);
    else if (type === effectTypes.PUT) runPutEffect(payload, currCb);
    else if (type === effectTypes.ALL) runAllEffect(payload, effectId, currCb);
    else if (type === effectTypes.RACE)
      runRaceEffect(payload, effectId, currCb);
    else if (type === effectTypes.CALL)
      runCallEffect(payload, effectId, currCb);
    else if (type === effectTypes.CPS) runCPSEffect(payload, currCb);
    else if (type === effectTypes.FORK)
      runForkEffect(payload, effectId, currCb);
    else if (type === effectTypes.JOIN) runJoinEffect(payload, currCb);
    else if (type === effectTypes.CANCEL) runCancelEffect(payload, currCb);
    else if (type === effectTypes.SELECT) runSelectEffect(payload, currCb);
    else if (type === effectTypes.ACTION_CHANNEL)
      runChannelEffect(payload, currCb);
    else if (type === effectTypes.FLUSH) runFlushEffect(payload, currCb);
    else if (type === effectTypes.CANCELLED)
      runCancelledEffect(payload, currCb);
    else if (type === effectTypes.GET_CONTEXT)
      runGetContextEffect(payload, currCb);
    else if (type === effectTypes.SET_CONTEXT)
      runSetContextEffect(payload, currCb);
    else currCb(effect);
  } else {
    // anything else returned as is
    currCb(effect);
  }
}
```

`runEffect`就是 saga 的核心函数，根据`iterator`每次 next 返回的结果（如果你在那行用了 effect），执行对应的处理函数。
可以看到除了官方指定的`effects`外，`iterator`是支持`yield promise` 和 `iterator`的。

## take 的实现

```js
function runTakeEffect({ channel = env.stdChannel, pattern, maybe }, cb) {
  const takeCb = input => {
    if (input instanceof Error) {
      cb(input, true);
      return;
    }
    if (isEnd(input) && !maybe) {
      cb(TERMINATE);
      return;
    }
    cb(input);
  };
  try {
    channel.take(takeCb, is.notUndef(pattern) ? matcher(pattern) : null);
  } catch (err) {
    cb(err, true);
    return;
  }
  cb.cancel = takeCb.cancel;
}
```

`take`是用来`watch` 指定 `pattern` 的`effect`,也就是说，当 saga 运行到这里的时候，会向 channel 里面注册一个 taker,

```js
chan.put = input => {
  if (input[SAGA_ACTION]) {
    put(input);
    return;
  }
  asap(() => put(input));
};
```

同时，这个过程是节流的，也就是`asap`，asap 是保证同一时间，注册和执行不能同时进行。具体的代码在下面

```js
const queue = [];
/**
  Variable to hold a counting semaphore
  - Incrementing adds a lock and puts the scheduler in a `suspended` state (if it's not
    already suspended)
  - Decrementing releases a lock. Zero locks puts the scheduler in a `released` state. This
    triggers flushing the queued tasks.
**/
let semaphore = 0;

/**
  Executes a task 'atomically'. Tasks scheduled during this execution will be queued
  and flushed after this task has finished (assuming the scheduler endup in a released
  state).
**/
function exec(task) {
  try {
    suspend();
    task();
  } finally {
    release();
  }
}

/**
  Executes or queues a task depending on the state of the scheduler (`suspended` or `released`)
**/
export function asap(task) {
  queue.push(task);

  if (!semaphore) {
    suspend();
    flush();
  }
}

/**
  Puts the scheduler in a `suspended` state. Scheduled tasks will be queued until the
  scheduler is released.
**/
export function suspend() {
  semaphore++;
}

/**
  Puts the scheduler in a `released` state.
**/
function release() {
  semaphore--;
}

/**
  Releases the current lock. Executes all queued tasks if the scheduler is in the released state.
**/
export function flush() {
  release();

  let task;
  while (!semaphore && (task = queue.shift()) !== undefined) {
    exec(task);
  }
}
```

`taker`会在`dispach`的时候，由`sagaMiddleware put action`进来,put 的过程中会循环遍历所有已经注册的 taker，并在调用后直接在数组中删除。也就是，take 采用的是单次注册，单次运行。

```js
put(input) {
      // TODO: should I check forbidden state here? 1 of them is even impossible
      // as we do not possibility of buffer here
      if (process.env.NODE_ENV === 'development') {
        check(input, is.notUndef, UNDEFINED_INPUT_ERROR)
      }

      if (closed) {
        return
      }

      if (isEnd(input)) {
        close()
        return
      }

      const takers = (currentTakers = nextTakers)
      for (let i = 0; i < takers.length; i++) {
        const taker = takers[i]
        if (taker[MATCH](input)) {
          taker.cancel()
          taker(input)
        }
      }
    },
```

`take`会返回当前的`action`,并开始执行`take`之后函数。`saga`就是从`take`开始的。

## call 的实现

```js
function runCallEffect({ context, fn, args }, effectId, cb) {
  let result;
  // catch synchronous failures; see #152
  try {
    result = fn.apply(context, args);
  } catch (error) {
    cb(error, true);
    return;
  }
  return is.promise(result)
    ? resolvePromise(result, cb)
    : is.iterator(result)
    ? resolveIterator(result, effectId, getMetaInfo(fn), cb)
    : cb(result);
}
```

`call`会执行当前指定的函数，并且函数的`this`绑定到了`saga`的`context`,并且在结果返回后再进行二次解析。最后将得出的属性通过`cb`传递给`iterator`。

## put 的实现

```js
function runPutEffect({ channel, action, resolve }, cb) {
  asap(() => {
    let result;
    try {
      result = (channel ? channel.put : env.dispatch)(action);
    } catch (error) {
      cb(error, true);
      return;
    }

    if (resolve && is.promise(result)) {
      resolvePromise(result, cb);
    } else {
      cb(result);
    }
  });
  // Put effects are non cancellables
}
```

如果我们没有传入`channel`，put 就会直接使用`redux`的`dispatch`来派发 action。
