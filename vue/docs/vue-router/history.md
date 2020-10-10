VueRouter对路由模式进行了拆分封装，核心逻辑放在`src/history/base.js`中，根据mode的不同分别定义了三个路由模式的实现和操作API，我们分别分析

## baseHistory

history基类，拥有最重要的`transitionTo`和`confirmTransition`两个方法，实现了路径切换的核心功能（包括匹配路由、执行导航守卫、切换当前路由、执行回调，监听前进后退事件）

定义了供子类继承重载的6个方法，是具体处理URL的API
```
+go: (n: number) => void
+push: (loc: RawLocation, onComplete?: Function, onAbort?: Function) => void
+replace: (
  loc: RawLocation,
  onComplete?: Function,
  onAbort?: Function
) => void
+ensureURL: (push?: boolean) => void
+getCurrentLocation: () => string
+setupListeners: Function
```

## HashHistory

hash模式我们很早就接触过，在原来实现楼层导航时，我们通过a标签指定href属性，以`href='#xxx'`的形式定位到页面某个具体的位置，它同样可以用来作为路由，我们通过不同的hash值区分不同的页面，通过监听`hashchange`事件来处理hash的变化

在VueRouter中，对于hash模式的实现首先判断是否支持`pushState`，如果支持则使用该方法改变URL，否则直接操作`window.location.hash`

## HTML5History

后面出现了history模式，它带来了`pushState`、`push`、`replace`、`go`等一系列API，专门用来操作浏览器的浏览历史记录，浏览器的前进后退事件通过`popstate`监听

在VueRouter中，对于history模式的实现其实就是操作这些API

## AbstractHistory

对于非浏览器环境，我们无法使用`window.history`，所以我们需要自己维护一个历史记录栈

VueRouter定义了一个`stack`数组保存历史url，同时使用`index`表示当前的url，通过栈模拟类似history的行为，完成路由的跳转
