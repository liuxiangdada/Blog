导航守卫是VueRouter提供的在路径切换的不同时机插入代码的钩子函数，我们编写钩子函数时，VueRouter已经为我们提供了三个参数`to`、`from`、`next`，`to`和`from`分别表示将要进入的路径和将要离开的路径，而`next`则是一个终止函数，当我们完成自定义代码时调用该方法来`resolve`这个钩子

## 守卫的触发时机

当我们使用`router-link`或者`this.$router.push`来切换路径时，最终调用的都是`router.history.transitionTo`来切换的，我们首先拿到待切换的路径，然后执行`confirmTransition`方法，该方法内部定义了`runQueue`和`iterator`方法来实现一个队列执行流程

队列中我们首先执行组件定义的`beforeRouteLeave`钩子，接着执行`this.router.beforeHooks`，然后执行组件的`beforeRouteUpdate`钩子，接着执行路由中定义的`beforeEnter`钩子，最后解析异步组件
```
const queue: Array<?NavigationGuard> = [].concat(
  // in-component leave guards
  extractLeaveGuards(deactivated),
  // global before hooks
  this.router.beforeHooks,
  // in-component update hooks
  extractUpdateHooks(updated),
  // in-config enter guards
  activated.map(m => m.beforeEnter),
  // async components
  resolveAsyncComponents(activated)
)
```

在`runQueue`执行完所有钩子后，就触发回调，回调中又新建了一个队列，包含了组件的`beforeRouteEnter`钩子和全局的`beforeResolve`钩子
```
runQueue(queue, iterator, () => {
  const postEnterCbs = []
  const isValid = () => this.current === route
  // wait until async components are resolved before
  // extracting in-component enter guards
  const enterGuards = extractEnterGuards(activated, postEnterCbs, isValid)
  const queue = enterGuards.concat(this.router.resolveHooks)
  runQueue(queue, iterator, () => {
    if (this.pending !== route) {
      return abort(createNavigationCancelledError(current, route))
    }
    this.pending = null
    onComplete(route)
    if (this.router.app) {
      this.router.app.$nextTick(() => {
        postEnterCbs.forEach(cb => {
          cb()
        })
      })
    }
  })
})
```

上面的第二个`runQueue`执行完后，我们从`extractEnterGuards`拿到了`beforeRouteEnter`钩子中next参数的回调函数并放到`postEnterCbs`中，然后在nextTick执行回调，确保我们能够拿到vm实例，这让我们可以实现这样的功能：
```
const Foo = {
  template: '<div>foo</div>',
  beforeRouterEnter (to, from, next) {
    next(vm => {
      // 拿到vm实例并处理
    })
  }
}
```

最后，我们调用`onComplete(route)`方法，实际上是执行下面的代码：
```
() => {
  this.updateRoute(route)
  onComplete && onComplete(route)
  this.ensureURL()
  this.router.afterHooks.forEach(hook => {
    hook && hook(route, prev)
  })

  // fire ready cbs once
  if (!this.ready) {
    this.ready = true
    this.readyCbs.forEach(cb => {
      cb(route)
    })
  }
},
```

在`updateRoute`方法中我们确认导航，切换路径
```
this.current = route
this.cb && this.cb(route)
```

这里的`this.cb`实际上是在`init`方法中调用`history.listen`添加的回调
```
listen (cb: Function) {
  this.cb = cb
}

history.listen(route => {
  this.apps.forEach(app => {
    app._route = route
  })
})
```

所以会对当前实例的`$route`重新赋值，从而触发视图的重新渲染

最后，我们调用全局的`afterEach`钩子
```
this.router.afterHooks.forEach(hook => {
  hook && hook(route, prev)
})
```
