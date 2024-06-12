---
title: Redux为什么需要中间件来处理异步数据流？
date: 2020-10-11
updated: 2020-10-11
issueid: 50
tags:
- 前端
---
我们明明可以在async之后直接调用dispatch，为什么又要多此一举引入中间件呢？

[为什么需要中间件来处理Redux异步数据流？](https://stackoverflow.com/questions/34570758/why-do-we-need-middleware-for-async-flow-in-redux)

## 如果异步之后直接Dispatch

```javascript
// action creator
function loadData(dispatch, userId) { // needs to dispatch, so it is first argument
  return fetch(`http://data.com/${userId}`)
    .then(res => res.json())f
    .then(
      data => dispatch({ type: 'LOAD_DATA_SUCCESS', data }),
      err => dispatch({ type: 'LOAD_DATA_FAILURE', err })
    );
}

// component
componentWillMount() {
  loadData(this.props.dispatch, this.props.userId); // don't forget to pass dispatch
}
```

 

## 如果使用 `react-thunk`
```js
// action creator
function loadData(userId) {
  return dispatch => fetch(`http://data.com/${userId}`) // Redux Thunk handles these
    .then(res => res.json())
    .then(
      data => dispatch({ type: 'LOAD_DATA_SUCCESS', data }),
      err => dispatch({ type: 'LOAD_DATA_FAILURE', err })
    );
}

// component
componentWillMount() {
  this.props.dispatch(loadData(this.props.userId)); // dispatch like you usually do
}
```

可以看到，`thunk` 只是更改了你的写法。因为不使用 `thunk` 你需要每次传递给异步函数 `dispatch`，使用 `thunk` 后在组件中就可以调用 `dispatch`，组件不再需要关注 `dispatch` 派发的函数是不是异步，组件只是发出一个请求而已。组件connect的数据通过 `dispatch => reducer` 流程最后会触发更新。

## Redux-thunk出现是为了解决什么问题？
优雅地进行异步调用，但thunk并不能终止请求，比如component unmounted 之后API调用结束这时会产生副作用，容易导致内存泄漏。
`react saga`就是为解决上述问题而生的，当然，后面再说 :)

[How to dispatch a Redux action with a timeout?](https://stackoverflow.com/questions/35411423/how-to-dispatch-a-redux-action-with-a-timeout/35415559#35415559)