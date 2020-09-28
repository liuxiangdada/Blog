## 安装

Vuex在浏览器版本不需要安装，直接使用就行，这是因为在Store类的构造器中会检测是否是通过`script`引用的并自动安装

而在npm版本，则需要在使用前使用`Vue.use`进行插件安装
```
import Vuex from 'vuex'

Vue.use(Vuex)
```

`use`方法的内部是调用`Vuex`对象的`install`方法，首先将`Vue`构造函数传入，如果是`2.x`版本以上的Vue，就在`beforeCreate`钩子中获取传入`vm.options`中的`store`对象，赋值给`vm.$store`，如何是组件，则向上寻找父组件的`$store`
```
const store = new Vuex.Store({
  state: {
    count: 0
  }
})

new Vue({
  el: '#app',
  store
})
```

## Store仓库的构建

在Vue实例化之前，我们就已经构建好了Vuex仓库`store`并把它传入Vue构造函数，构建仓库时调用了`Store`这个类，该类主要做了三件事

- 初始化数据
- 注册模块
- 添加响应式

### 初始化数据

```
this._committing = false
```

一个`commit`标识，所有Vuex的响应式数据都会被监听其修改并判断这个标识是否为`true`，如果不满足，提示不能直接修改，内部在执行`commit`方法提交时，会调用`_withCommit`方法绑定的回调，该回调先修改`_committing`，再修改数据

```
this._actions = Object.create(null)
this._mutations = Object.create(null)
this._wrappedGetters = Object.create(null)
```

这三个对象中存放着所有仓库（包括子模块）中的操作方法`getters、mutations、actions`

```
this._modules = new ModuleCollection(options)
```

这个对象中存放着所有模块的数据，模块构建成了一棵树，所以`this._modules`对象只有一个`root`属性，代表根模块

在`ModuleCollection`类中，我们根据`modules`属性递归的收集模块，模块的初始化过程主要在`Module`类，在Module类里定义了一些基本属性如`_children`和`_rawModule`

### 注册模块

拿到我们传入的`modules`属性中的信息后，我们就拥有每个模块的数据和方法，这些的模块就好像一个个小仓库，共同构成了`root`这个大仓库

我们注意到一个问题，就是我们该如何区分每个模块中的数据和方法，因为上面我们知道，所有的数据都可以通过`state`来访问，通过`commit`和`action`来修改，但我们在模块内部时并没有感知到这一点，我们并不需要指定某个模块而是直接就能操作本模块的数据，这是因为Vuex已经帮我们做了数据、方法的映射

我们定义在根模块的数据会存放在`this._modules.root.state`中，而模块中的数据会在各模块的state中，所以我们以模块名为标识，把数据收集到`this._module.root.state`中
```
store._modules.root.state: {
  count: 0,
  a: {
    count: 0,
    nestA: {
      count: 0
    }
  }
}
```

从全局模块中取出本地模块被称为本地化，要实现本地化就要对操作数据的各种方法进行重载，思路是通过模块名取到命名空间`namespace`，手动在各个操作数据的方法中拼接出完整的方法名，再执行方法，具体实现在`makeLocalContext`方法：
```
// 简化版
const local = {
  dispatch: type => {
    type = namespace + type
    store.dispatch(type)
  },
  commit: type => {
    type = namespace + type
    store.commit(type)
  }
}

Object.defineProperties(local, {
  getters: {
    get: () => makeLocalGetters(store, namespace)
  },
  state: {
    get: () => getNestedState(store.state, path)
  }
})
```

最后我们根据`namespaced`属性，为模块中的数据方法加上命名空间，并存入上面初始化的`this._actions、this._mutations、this._wrappedGetters`三个对象中，我们在方法前面加上`name/`前缀用以区分不同的模块，方便后面调用；我们之所以能在结构赋值`commit`和`action`后仍能正常使用，是因为这些方法内部在注册时都硬绑定了当前store实例

值得注意的是，模块的`getter`、`commit`和`action`方法都能传入参数，并使用当前模块的各个数据方法，正是因为前面本地化得到`local`后，作为参数传入了这些方法中
```
// mutation
function wrappedMutationHandler (payload) {
  handler.call(store, local.state, payload)
}

// action
function wrappedActionHandler (payload) {
  let res = handler.call(store, {
    dispatch: local.dispatch,
    commit: local.commit,
    getters: local.getters,
    state: local.state,
    rootGetters: store.getters,
    rootState: store.state
  }, payload)
  if (!isPromise(res)) {
    res = Promise.resolve(res)
  }
  if (store._devtoolHook) {
    return res.catch(err => {
      store._devtoolHook.emit('vuex:error', err)
      throw err
    })
  } else {
    return res
  }
}

// getter
function wrappedGetter (store) {
  return rawGetter(
    local.state, // local state
    local.getters, // local getters
    store.state, // root state
    store.getters // root getters
  )
}
```

### 添加响应式

我们在`resetStoreVM`方法中创建了一个新的Vue实例存放在`store._vm`中

通过把`this._modules.root.state`的数据引用传入`_vm._data.$$state`中，就能把state数据变为响应式，这样即使不通过具体的方法使用这些数据而是直接通过`this.$store.state`访问，在数据变更时，视图也会对应修改

在使用Vuex的`getters`时，是能够和Vue中的`computed`一样，只有当其依赖的state变化的时候才重新求值，正是因为我们把`wrappedGetters`作为`computed`映射到_vm上，这样内部的`computedWtacher`就能在state变化时重新求值
```
const computed = {}
forEachValue(wrappedGetters, (fn, key) => {
  computed[key] = fn(store)
  Object.defineProperty(store.getters, key, {
    get: () => store._vm[key]
  })
})

store._vm = new Vue({
  data: {
    $$state: state
  },
  computed
})

```