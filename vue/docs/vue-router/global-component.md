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
