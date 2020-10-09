前面我们提到在`VueRouter`类的构造函数中会生成匹配器`matcher`，这样我们就能根据传入的location找出将要跳转的路由，完成路由的核心功能

下面我们就分析一下matcher的构建过程

## ceaterMatcher

调用该方法传入`routes`和`router`作为参数，最后暴露`match`和`addRoutes`方法，此外，我们还根据routes配置调用`createRouteMap`生成了`pathList`、`pathMap`和`nameMap`三个表
```
const { pathList, pathMap, nameMap } = createRouteMap(routes)
```

这三个表中，`pathList`中保存着用户配置的路径，`pathMap`中保存着从路径到`RouteRecord`的映射，`nameMap`中保存着从name到`RouteRecord`的映射

这个`RouteRecord`是什么呢，我们阅读源码可知，它是通过遍历routes中每个route并调用`addRouteRecord`方法生成的，在`flow/declarations.js`中找到其数据结构如下：
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

## match

## addRoutes
