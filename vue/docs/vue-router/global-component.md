## router-link

`router-link`是VueRouter内置的全局组件，和HTML中的`a`标签类似（其实组件内部实现时的默认承载标签也是a），用作路由导航，之所以封装一个全新的组件，一个原因是丰富原来链接标签的功能，另一个是统一浏览器在不同路由模式下的行为

`router-link`的工作原理是重写了`render`函数，我们通过组件的`to`参数，获得需要导航的路径原始信息，然后调用`router.resolve`方法解析出目标路径`route`和最终跳转的`href`，接着判断当前路由和组件要跳转的路径是否相同，添加`activeClass`类和`exactActiveClass`类，接着调用`router.push`或`router.replace`封装一个点击事件`handler`，最后调用`h`方法创建一个组件VNode并返回

## router-view
