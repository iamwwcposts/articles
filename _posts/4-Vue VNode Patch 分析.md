---
date: 2019-09-11
updated: 2019-09-11
issueid: 4
tags:
title: Vue VNode Patch 分析
---
这里记录一下Vue的 Virtual DOM 比较过程
来自于 `cn.vuejs.org` `patchVnode` 函数断点

当我们对于data进行修改之后会产生新的 VDOM 集合
这里是 vnode

oldVnode则代表修改之前的 VDOM

```js
function patchVnode (
    oldVnode,
    vnode,
    insertedVnodeQueue,
    ownerArray,
    index,
    removeOnly
  ) {
    if (oldVnode === vnode) {
      return
    }

    if (isDef(vnode.elm) && isDef(ownerArray)) {
      // clone reused vnode
      vnode = ownerArray[index] = cloneVNode(vnode)
    }

    const elm = vnode.elm = oldVnode.elm

    if (isTrue(oldVnode.isAsyncPlaceholder)) {
      if (isDef(vnode.asyncFactory.resolved)) {
        hydrate(oldVnode.elm, vnode, insertedVnodeQueue)
      } else {
        vnode.isAsyncPlaceholder = true
      }
      return
    }

    // reuse element for static trees.
    // note we only do this if the vnode is cloned -
    // if the new node is not cloned it means the render functions have been
    // reset by the hot-reload-api and we need to do a proper re-render.
    if (isTrue(vnode.isStatic) &&
      isTrue(oldVnode.isStatic) &&
      vnode.key === oldVnode.key &&
      (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
    ) {
      vnode.componentInstance = oldVnode.componentInstance
      return
    }

    let i
    const data = vnode.data
    if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
      i(oldVnode, vnode)
    }

    const oldCh = oldVnode.children
    const ch = vnode.children
    if (isDef(data) && isPatchable(vnode)) {
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
      if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
    }
    if (isUndef(vnode.text)) {
      if (isDef(oldCh) && isDef(ch)) {
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
      } else if (isDef(ch)) {
        if (process.env.NODE_ENV !== 'production') {
          checkDuplicateKeys(ch)
        }
        if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
      } else if (isDef(oldCh)) {
        removeVnodes(elm, oldCh, 0, oldCh.length - 1)
      } else if (isDef(oldVnode.text)) {
        nodeOps.setTextContent(elm, '')
      }
    } else if (oldVnode.text !== vnode.text) {
      nodeOps.setTextContent(elm, vnode.text)
    }
    if (isDef(data)) {
      if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
    }
  }
```

Vue 会在 `updateChildren` 中比较 `VNode`
每次调用`sameVnode`，当节点相同时调用 `patchVNode`
`patchVNode`中会再次调用 `updateChildren` 进行更新

这是一个间接的递归

有个疑问
Vue一直再调用 updateChildren

既然是diff，应该会生成一个差异补丁
但 updateChildren 并未有返回值，那么patch之后最终如何渲染到UI上的？

----------------------------------------

Vue在模板编译完成之后会生成组件的渲染函数

这个渲染函数调用之后会返回VNode节点

```js
vnode = render.call(vm._renderProxy, vm.$createElement);
```


```js
updateComponent = function () {
    vm._update(vm._render(), hydrating);
};
```

`vm_render()` 调用render function 获得VNode，然后进行_update，根据VNode将DOM渲染到HTML中

也即，`render` 获得 `VDOM`， `_update` 根据 `VDOM` 来渲染 `DOM`

-----------------------------------

如果没记错的话 target stack 是 Vue2 中才引入的机制，而 Vue1 中则是仅靠 Dep.target 来进行依赖收集的。根据我自己对 Vue1 和 Vue2 差异的理解，引入 target stack 的原因在于 Vue2 使用了新的视图更新方式。

具体来说，vue1 视图更新采用的是细粒度绑定的方式，而 vue2 采取的是 virtual DOM 的方式。举个例子来说可能比较容易理解，对于下面的模版：
```
<!-- root -->
<div>
{{ a }}
<my :text="b"></my>
{{ c }}
<div>

<!-- component my -->
<span>{{ b }}</span>
```

Vue1 的处理方式可以简化理解为：

```
watch(for a) -> directive(update {{ a }})
watch(for b) -> directive(update {{ b }})
watch(for c) -> directive(update {{ c }})
```
由于是数据到 DOM 操作操作指令的细粒度绑定，所以不论是指令还是 watcher 都是原子化的。对于上面的模版，在处理完 {{ a }} 的视图绑定后，创建新的 vue 实例 my 并且处理 {{ b }} 的视图绑定，随后继续处理 {{ c }}的绑定。

而在 Vue2 中情况就完全不同，视图被抽象为一个 render 函数，一个 render 函数只会生成一个 watcher，其处理机制可以简化理解为：

```
renderRoot () {
	...
	renderMy ()
	...
}
```
可以看到在 Vue2 中组件数的结构在视图渲染时就映射为 render 函数的嵌套调用，有嵌套调用就会有调用栈。当 evaluate root 时，调用到 my 的 render 函数，此时就需要中断 root 而进行 my 的 evaluate，当 my 的 evaluate 结束后 root 将会继续进行，这就是 target stack 的意义。

未完待续。。