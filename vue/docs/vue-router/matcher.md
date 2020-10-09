前面我们提到在`VueRouter`类的构造方法中会生成匹配器`matcher`，这样我们就能根据传入的location找出将要跳转的路由，完成路由的核心功能

下面我们就分析一下matcher的构建过程

## ceaterMatcher

调用该方法传入`routes`和`router`作为参数，最后暴露`match`和`addRoutes`方法，此外，我们还根据routes配置调用`createRouteMap`生成了`pathList`、`pathMap`和`nameMap`三个表
```
const { pathList, pathMap, nameMap } = createRouteMap(routes)
```

这三个表中，`pathList`中保存着用户提供的路径，`pathMap`中保存着以路径为键，`Record`为值的一组映射，`nameMap`中保存着以name为键，`Record`为值的一组映射

这个`Record`是什么呢，我们阅读源码可知，它是通过遍历routes中每个route并调用`addRouteRecord`方法生成，其数据结构如下
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

我们在定义routes时，时常会定义`children`，这样会导致结构变得复杂，所以我们在解析时需要递归的构建出一棵`Record`树，方便我们使用，常见的routes定义如下
```
const routes = [
  {
    path: '/foo',
    component: Foo,
    children: [
      {
        path: 'baz',
        component: Baz
      }
    ]
  },
  { path: '/bar', alias: '/baa', component: Bar }
]
```



## match

## addRoutes