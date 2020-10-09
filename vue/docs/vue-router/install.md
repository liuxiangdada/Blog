如果我们需要使用`Vue-Router`，则需要在`main.js`中安装插件，实例化router对象并在`new Vue`时作为options注入

## 注入

安装的方法由`install`实现，该方法实现在vuex源码的`src/install.js`路径下，安装过程主要干了以下几件事

1.防止重复安装
```
if (install.installed && _Vue === Vue) return
install.installed = true

_Vue = Vue
```

2.使用`Vue.mixin`方法全局注入`beforeCreated`和`destroyed`钩子

a.在`beforeCreated`钩子中，首先判断有没有在`new Vue`时传入router对象，有的话我们使用`this._routerRoot`保存当前实例，这么做的好处是我们在`Vue-Router`源码中就能用这个属性代指vm实例，语义化更强
```
this._routerRoot = this
```

接着我们保存router对象到`this._router`并调用`init`方法初始化
```
this._router = this.$options.router
this._router.init(this)
```

接着使用`defineReactive`方法将`this._route`的值赋为`this._router.history.current`并设为响应式对象
```
Vue.util.defineReactive(this, '_route', this._router.history.current)
```

最后调用`registerInstance`方法注册当前实例，作用后面讲

b.在`destroyed`钩子中调用`registerInstance`方法将当前实例移除

3.给`Vue.prototype`绑定`$route`和`$router`两个全局对象

4.使用`Vue.component`方法全局注册`RouterLink`和`RouterView`两个组件

5.设置了组件内`beforeRouteEnter`、`beforeRouteLeave`和`beforeRouteUpdate`三个钩子的合并策略

这三个钩子的合并策略和Vue的created钩子的合并策略保持一致

## 实例化

在安装后我们需要实例化router对象，实例化发生在`src/index.js`中定义的`VueRouter`类中，该类定义一些内部属性和暴露的参数，同时定义了一些内置方法和API，使得我们可以在`this.$router`中使用API来控制路由

在构造函数中，主要初始化了一些属性
```
this.app = null
this.apps = []
this.options = options
this.beforeHooks = []
this.resolveHooks = []
this.afterHooks = []
this.matcher = createMatcher(options.routes || [], this)
```

这里最重要的就是根据用户传入的`routes`路由配置生成了路由匹配器`matcher`，它的创建过程和作用我会单独解释

接着我们根据`options.mode`参数决定使用哪种路由模式，这里做了一个降级优化，如果浏览器不支持`PushState`，则会退化到使用`hash`模式，如果不在浏览器中则使用`abstract`模式
```
let mode = options.mode || 'hash'
this.fallback =
  mode === 'history' && !supportsPushState && options.fallback !== false
if (this.fallback) {
  mode = 'hash'
}
if (!inBrowser) {
  mode = 'abstract'
}
this.mode = mode
```

接着我们定义了一堆方法，我们分为内置方法和API来一一解释

内置方法
- init，初始化router，做一些前置工作，后面单独讲
- match，匹配路由，主要是根据当前路径和传入的location匹配出要跳转到的路由

暴露方法
- beforeEach、beforeResolve、afterEach，三个全局钩子，在路由跳转的不同时机触发
- push、replace、go、back、forward，导航方法，控制路由前进后退
- getMatchedComponents，根据传入位置找出匹配路由中设置的组件数组
- resolve，解析目标location的位置，返回具体的路由信息
- addRoutes，给用户提供动态添加路由的一种方法
- onReady，注册一个回调，在完成初始导航时调用，这意味着它能处理异步钩子和异步组件
- onError，注册一个回调，在导航发生错误时调用

