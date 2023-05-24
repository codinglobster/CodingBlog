---
title: dva-source-code-read
description: "dva源码试探阅读"
pubDatetime: 2018-08-08 16:51:36
author: codinglobster
ogImage: "dva.png"
tags:
  - react
  - redux
---

## 前言

从接触`react`开始就是使用`dva`作为入门框架来学习的，她封装完整，简约的 api 让新入门的新手也能跟着`十二步完成curd-demo`。直到最近完成了 dva 底层框架的代码阅读后，我终于拾起了荒废已久的`dva`源码阅读计划，看看她是怎么通过`model`来实现 dva 框架的整合。

> dva 是守望先锋中的一名英雄，是一名操作可飞行机甲的韩国人气少女偶像，拥有极强的机动性和防御力，是冲散敌人后排的强力英雄。阿里果然隐藏着一众死宅程序员。

## 流程图

![dva](./dva-process.png)
修改浏览器缩放比率查看大图

## 从 dva2 开始

`dva2.0`是 17 年底发布的，不仅更新了`react-router4`,更提供了许多优化，如`dispatch => promise`、effect 执行前后后会派发`type/@@start、type/@@end`事件，可以借用此特性实现`put`代码的同步执行。
而且从 2.0 开始，`dva`对核心数据流处理部分进行了拆分，称为`dva-core`,这一部分只实现`redux`和`redux-saga`的整合，以及`model api`的实现。

## create

```js
export function create(hooksAndOpts = {}, createOpts = {}) {
  const { initialReducer, setupApp = noop } = createOpts;

  const plugin = new Plugin();
  plugin.use(filterHooks(hooksAndOpts));

  const app = {
    _models: [prefixNamespace({ ...dvaModel })],
    _store: null,
    _plugin: plugin,
    use: plugin.use.bind(plugin),
    model,
    start,
  };
  return app;
}
```

`hooksAndOpts`是我们初始化时传入的`hooks`,而`createOpts`则是框架内部传入的参数（如 react-router 的 middleware 和 reducer）,
`create`的作用其实是初始化我们常用的`app`对象。
`_models`会存贮我们定义的所有`model`,通过`model`方法来注册，并在注册的时候为`reducer`和`effect`添加`namespace`，方便日后 dispatch.
`_store`则是在`start`时会将生成的 store 注册到这里。
`_plugin`存储的是`dva`支持的所有 hook，除了`extraEnhancer`和`_handleActions`其它都是数组形式的，支持多次注册。
`use`则是注册 plugin 的作用，dva 的 plugin 实际上是 hooks 的组合对象，如`dva-loading`是由`extraReducer`和`onEffect`组合而成的。

## model

```js
function model(m) {
  if (process.env.NODE_ENV !== "production") {
    checkModel(m, app._models);
  }
  const prefixedModel = prefixNamespace({ ...m });
  app._models.push(prefixedModel);
  return prefixedModel;
}
```

model 的作用已经说过，将传入的 model 注册到 app 的\_model 中。

## start

start 是 dva-core 的核心，完成所有的框架注册。

```js
// Global error handler
const onError = (err, extension) => {
  if (err) {
    if (typeof err === "string") err = new Error(err);
    err.preventDefault = () => {
      err._dontReject = true;
    };
    plugin.apply("onError", err => {
      throw new Error(err.stack || err);
    })(err, app._store.dispatch, extension);
  }
};
```

第一步完成 onError 的注册，如果有自定义规则会进行整合

第二部提取 reducer 和 effect

```js
const sagas = [];
const reducers = { ...initialReducer };
for (const m of app._models) {
  reducers[m.namespace] = getReducer(
    m.reducers,
    m.state,
    plugin._handleActions
  );
  if (m.effects)
    sagas.push(app._getSaga(m.effects, m, onError, plugin.get("onEffect")));
}
```

getReducer 的过程中如果没有使用自定义钩子则会使用系统默认的规则。

```js
function handleAction(actionType, reducer = identify) {
  return (state, action) => {
    const { type } = action;
    invariant(type, "dispatch: action should be a plain Object with type");
    if (actionType === type) {
      return reducer(state, action);
    }
    return state;
  };
}

function reduceReducers(...reducers) {
  return (previous, current) =>
    reducers.reduce((p, r) => r(p, current), previous);
}

function handleActions(handlers, defaultState) {
  const reducers = Object.keys(handlers).map(type =>
    handleAction(type, handlers[type])
  );
  const reducer = reduceReducers(...reducers);
  return (state = defaultState, action) => reducer(state, action);
}
```

reducer 先是被进行了一层封装，只有 type 相同时才会进行执行，后是进行了整合，将多个 reducer 合并为了一个嵌套调用的 reducer。
最后注册到外层的 reducer[namespace]中。

getSaga 则进行了更多深层的操作。

```js
export default function getSaga(effects, model, onError, onEffect) {
  return function* () {
    for (const key in effects) {
      if (Object.prototype.hasOwnProperty.call(effects, key)) {
        const watcher = getWatcher(key, effects[key], model, onError, onEffect);
        const task = yield sagaEffects.fork(watcher);
        yield sagaEffects.fork(function* () {
          yield sagaEffects.take(`${model.namespace}/@@CANCEL_EFFECTS`);
          yield sagaEffects.cancel(task);
        });
      }
    }
  };
}
```

可以看到是返回了匿名的生成器函数，fork 了 watcher，然后注册了一个取消 watcher 的 watcher。

```js
function getWatcher(key, _effect, model, onError, onEffect) {
  let effect = _effect;
  let type = 'takeEvery';
  let ms;

  ...

  switch (type) {
    case 'watcher':
      return sagaWithCatch;
    case 'takeLatest':
      return function*() {
        yield takeLatest(key, sagaWithOnEffect);
      };
    case 'throttle':
      return function*() {
        yield throttle(ms, key, sagaWithOnEffect);
      };
    default:
      return function*() {
        yield takeEvery(key, sagaWithOnEffect);
      };
  }
}
```

getWatcher 的作用是根据传入的参数来决定监听的方式。来看一下 sagaWithOnEffect 的具体实现。

```js
function* sagaWithCatch(...args) {
  const { __dva_resolve: resolve = noop, __dva_reject: reject = noop } =
    args.length > 0 ? args[0] : {};
  try {
    yield sagaEffects.put({ type: `${key}${NAMESPACE_SEP}@@start` });
    const ret = yield effect(...args.concat(createEffects(model)));
    yield sagaEffects.put({ type: `${key}${NAMESPACE_SEP}@@end` });
    resolve(ret);
  } catch (e) {
    onError(e, {
      key,
      effectArgs: args,
    });
    if (!e._dontReject) {
      reject(e);
    }
  }
}

const sagaWithOnEffect = applyOnEffect(onEffect, sagaWithCatch, model, key);
```

在被执行之前，sagaWithCatch 需要被 onEffect 函数进行初次处理，而 sagaWithCatch 则比较复杂，会接收 resole 和 reject，这是个 promiseMiddleware 做处理接收到的。后面再来讨论。
可以看到 saga 被执行前后会分别派发对应的`start、end`事件，并将 saga 返回的结果通过 resolve 传入。

第三步，创建 store

```js
const store = (app._store = createStore({
  // eslint-disable-line
  reducers: createReducer(),
  initialState: hooksAndOpts.initialState || {},
  plugin,
  createOpts,
  sagaMiddleware,
  promiseMiddleware,
}));
```

createReducer 是将 onReducer 和 extraReducer 进行了整合，并传递给 redux,
enhancer 部分则是将 sagaMiddleware 和 promiseMiddleware 进行了整合，同时还有外部传入的 middleware。

第四步，redux 订阅注册

```js
const listeners = plugin.get("onStateChange");
for (const listener of listeners) {
  store.subscribe(() => {
    listener(store.getState());
  });
}
```

第五步，注册杂项

```js
// Run sagas
sagas.forEach(sagaMiddleware.run);

// Setup app
setupApp(app);

// Run subscriptions
const unlisteners = {};
for (const model of this._models) {
  if (model.subscriptions) {
    unlisteners[model.namespace] = runSubscription(
      model.subscriptions,
      model,
      app,
      onError
    );
  }
}
```

第六步，更新 model 和 unmodel 的 hook

```js
function injectModel(createReducer, onError, unlisteners, m) {
  m = model(m);

  const store = app._store;
  store.asyncReducers[m.namespace] = getReducer(
    m.reducers,
    m.state,
    plugin._handleActions
  );
  store.replaceReducer(createReducer());
  if (m.effects) {
    store.runSaga(app._getSaga(m.effects, m, onError, plugin.get("onEffect")));
  }
  if (m.subscriptions) {
    unlisteners[m.namespace] = runSubscription(
      m.subscriptions,
      m,
      app,
      onError
    );
  }
}
```

因为项目已经 start,所以要动态添加 model 不仅要变化 reducer 还有 effect，所以需要重新注册。

至此所有的注册流程已经完成，但是我们还有 promiseMiddleware 没有讲！！！

## promiseMiddleware

```js
export default function createPromiseMiddleware(app) {
  return () => next => action => {
    const { type } = action;
    if (isEffect(type)) {
      return new Promise((resolve, reject) => {
        next({
          __dva_resolve: resolve,
          __dva_reject: reject,
          ...action,
        });
      });
    } else {
      return next(action);
    }
  };

  function isEffect(type) {
    if (!type || typeof type !== "string") return false;
    const [namespace] = type.split(NAMESPACE_SEP);
    const model = app._models.filter(m => m.namespace === namespace)[0];
    if (model) {
      if (model.effects && model.effects[type]) {
        return true;
      }
    }

    return false;
  }
}
```

promiseMiddleware 将传入的 action 进行判断是否在 app 中注册对应 effect，如果注册，则将 action 变为 promise 形式，所以我们在 dispatch 后得到的其实是 promise 对象，从而可以结合 effect 处特殊的处理方式，直接通过.then 得到结果，从而改变 redux 的数据处理方式。
