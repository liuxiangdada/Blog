## router-link

`router-link`是VueRouter内置的全局组件，和HTML中的`a`标签类似（其实组件内部实现时的默认承载标签也是a），用作路由导航，之所以封装一个全新的组件，一个原因是丰富原来链接标签的功能，另一个是统一浏览器在不同路由模式下的行为

`router-link`的工作原理是重写了`render`函数，我们通过组件的`to`参数，获得需要导航的路径原始信息，然后调用`router.resolve`方法解析出目标路径`route`和最终跳转的`href`，接着判断当前路由和组件要跳转的路径是否相同或者是否存在包含关系，添加`activeClass`类和`exactActiveClass`类，接着调用`router.push`或`router.replace`封装一个点击事件`handler`，最后调用`h`方法创建一个组件VNode并返回

有两个细节要注意，一个是绑定的事件做了一些前置处理
```
function guardEvent (e) {
  // don't redirect with control keys
  if (e.metaKey || e.altKey || e.ctrlKey || e.shiftKey) return
  // don't redirect when preventDefault called
  if (e.defaultPrevented) return
  // don't redirect on right click
  if (e.button !== undefined && e.button !== 0) return
  // don't redirect if `target="_blank"`
  if (e.currentTarget && e.currentTarget.getAttribute) {
    const target = e.currentTarget.getAttribute('target')
    if (/\b_blank\b/i.test(target)) return
  }
  // this may be a Weex event which doesn't have this method
  if (e.preventDefault) {
    e.preventDefault()
  }
  return true
}
```

另一个细节是`tag`，在`router-link`组件中我们是可以通过传入tag参数来指定其他标签作为渲染节点的，如果我们指定的标签不是a标签或者没有`href`属性，就会在该节点内部寻找a标签，并把事件和href属性绑定到这个节点上
```
const a = findAnchor(this.$slots.default)

function findAnchor (children) {
  if (children) {
    let child
    for (let i = 0; i < children.length; i++) {
      child = children[i]
      if (child.tag === 'a') {
        return child
      }
      if (child.children && (child = findAnchor(child.children))) {
        return child
      }
    }
  }
}
```

## router-view

路由的跳转最终展现在页面上就是组件的变化，所以切换路由实质上是根据路径找到路由配置中的组件，然后渲染到`router-view`中，我们有两个问题需要解决

1.路径切换后，为什么能触发页面重新渲染呢？

在前面我们提到，VueRouter会注入一个beforeCreate钩子，其中有这么一段代码：
```
Vue.util.defineReactive(this, '_route', this._router.history.current)
```

它把`this._route`设置为响应式对象，当然一般我们不会直接引用这个属性因为它是内置属性，通常引用`this.$route`，因为：
```
Object.defineProperty(Vue.prototype, '$route', {
  get () { return this._routerRoot._route }
})
```
这样当我们访问`this.$route`时就会收集依赖，依赖的收集在`router-view`组件的render函数中
```
const route = parent.$route
```

完成路径切换后，我们会执行回调
```
  history.listen(route => {
    this.apps.forEach(app => {
      app._route = route
    })
  })
```

这里会对`this._route`重新赋值，这样就能触发重新渲染了

2.路由切换后是如何找到对应的组件的，嵌套路由又是如何处理的？

路由切换后我们通过`this.$route`拿到当前路由对象，其`matched`属性中就存放着组件对象配置，如果不是嵌套`router-view`，我们就可以取第一个组件进行渲染，这里要注意，由于命名视图的存在，我们在取组件时可以传入name指定渲染某个组件，默认渲染名为`default`的组件

如果存在嵌套路由，说明我们在定义路由时在某条路由配置下提供了`children`属性，以此为基础会得到一棵`RouteRecord`树，在路由切换后，我们的matched经过`formatMatch`方法的处理就会递归拿到该匹配`RouteRecord`往上寻找的所有父路由`RouteRecord`并且顺序是先父后子（即最精确的匹配在栈的底部）
```
function formatMatch (record: ?RouteRecord): Array<RouteRecord> {
  const res = []
  while (record) {
    res.unshift(record)
    record = record.parent
  }
  return res
}
```

在`router-view`组件的渲染函数中，定义了data的`routerView`属性，它代表这个是一个RouterView组件，这样在渲染视图的过程中，如果再次发现`router-view`视图组件就会根据`routerView`属性向前寻找，找出当前`router-view`的嵌套深度并从`matched[depth]`拿到对应`RouteRecord`

之所以可以通过计数的方式来寻找嵌套路由对应的组件是因为我们在定义路由和渲染路由时的层级是一一对应的，只要你定义了多个层级的嵌套路由，在视图中就会按顺序出现多个`router-view`
```
// 路由配置
const routes = [
  {
    path: '/foo',
    component: Foo,
    children: [
      {
        path: 'bar',
        component: Bar
      }
    ]
  }
]

// 对应的html
const Foo = {
  template: '<div>' +
  '<router-link to="/foo/bar">Bar</router-link>' +
  '<br>' +
  '<router-view></router-view>' +
  '</div>'
}
```
