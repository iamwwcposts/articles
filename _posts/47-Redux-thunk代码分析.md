---
date: 2020-08-25
updated: 2020-08-25
issueid: 47
tags:
- 前端
title: Redux-thunk代码分析
---
下面是redux的applyMiddleware代码

```
export default function applyMiddleware(
  ...middlewares: Middleware[]
): StoreEnhancer<any> {
  return (createStore: StoreEnhancerStoreCreator) => <S, A extends AnyAction>(
    reducer: Reducer<S, A>,
    preloadedState?: PreloadedState<S>
  ) => {
    const store = createStore(reducer, preloadedState)
    let dispatch: Dispatch = () => {
      throw new Error(
        'Dispatching while constructing your middleware is not allowed. ' +
          'Other middleware would not be applied to this dispatch.'
      )
    }

    const middlewareAPI: MiddlewareAPI = {
      getState: store.getState,
      dispatch: (action, ...args) => dispatch(action, ...args)
    }
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose<typeof dispatch>(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```

下面就是Thunk这个中间件的代码

```
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => (next) => (action) => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }

    return next(action);
  };
}

const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;

export default thunk;
```

这个中间件返回了嵌套的函数闭包，其中下面的函数就是applyMiddlewares的 `store.dispatch`

```
(action) => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }

    return next(action);
  };
}
```

回忆一下，我们 `connect` 之后的组件通过 `dispatch` 调用，传递的

```
{
type: SET_MESSAGE,
payload: 'hello'
}
```

就是 `action` 的参数结构