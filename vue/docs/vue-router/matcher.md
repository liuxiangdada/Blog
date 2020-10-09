前面我们提到在`VueRouter`类的构造函数中会生成匹配器`matcher`，这样我们就能根据传入的location找出将要跳转的路由，完成路由的核心功能

下面我们就分析一下matcher的构建过程

## ceaterMatcher

调用该方法传入`routes`和`router`作为参数，最后暴露`match`和`addRoutes`方法，此外，我们还根据routes配置调用`createRouteMap`生成了`pathList`、`pathMap`和`nameMap`三个表
```
const { pathList, pathMap, nameMap } = createRouteMap(routes)
```

这三个表中，`pathList`中保存着用户配置的路径，`pathMap`中保存着从路径到`RouteRecord`的映射，`nameMap`中保存着从name到`RouteRecord`的映射

这个`RouteRecord`是什么呢，阅读源码可知，它是通过遍历routes中每个route并调用`addRouteRecord`方法生成的，在`flow/declarations.js`中找到其数据结构如下：
```
declare type RouteRecord = {
  path: string;
  regex: RouteRegExp;
  components: Dictionary<any>;
  instances: Dictionary<any>;
  name: ?string;
  parent: ?RouteRecord;
  redirect: ?RedirectOption;
  matchAs: ?string;
  beforeEnter: ?NavigationGuard;
  meta: any;
  props: boolean | Object | Function | Dictionary<boolean | Object | Function>;
}
```

这个`RouteRecord`可以简单理解为一个用来匹配对应组件的记录，在这之前，我们还需要理解`Location`和`Route`这两个概念

首先来看`Location`，这个对象定义在`flow/declarations.js`中
```
declare type Location = {
  _normalized?: boolean;
  name?: string;
  path?: string;
  hash?: string;
  query?: Dictionary<string>;
  params?: Dictionary<string>;
  append?: boolean;
  replace?: boolean;
}
```

可以看出，这个数据对象是用来描述url的各个属性的，跟`window.location`类似

然后我们看一下`Route`这个对象的定义：
```
declare type Route = {
  path: string;
  name: ?string;
  hash: string;
  query: Dictionary<string>;
  params: Dictionary<string>;
  fullPath: string;
  matched: Array<RouteRecord>;
  redirectedFrom?: string;
  meta?: any;
}
```

`Route`对象就是我们在VueRouter插件中切换的路径对象，它大部分属性和`Location`对象相同，用来记录路径的信息，其中最重要的属性是`matched`，这个数组中存放着前面的`RouteRecord`

我们试着用这三个概念来描述一下路径切换的过程，首先我们编写路由配置`routes`，在`createMatcher`方法中构建出`RouteRecord`树，当我们要切换路径时，我们输入一个路径，该路径可以是字符串或者对象，但最终都会被规范成`Location`对象，我们根据`Location`对象和当前路径`current`，调用`match`方法就能匹配出新的路径，然后我们根据新路径中的`matched`数组中的`RouteRecord`，就能拿到组件配置，触发组件导航守卫，渲染出组件

## match

## addRoutes
