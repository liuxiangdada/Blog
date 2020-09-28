## 获取数据

### State

我们在组件中可以通过`this.$store.state`来访问数据，这是因为我们在`Store`类中定义了`state`的存取值函数
```
get state () {
  return this._vm._data.$$state
}

set state (v) {
  if (__DEV__) {
    assert(false, `use store.replaceState() to explicit replace store state.`)
  }
}
```

可以看到，我们实际上访问的是`this._vm._data.$$state`，而`$$state`是在生成_vm实例时，传入的`this._modules.root.state`，这样一一映射，我们最终拿到了我们定义的数据

针对模块内部的数据，Vuex在构建模块树时，会通过`Vue.set(parentState, moduleName, module.state)`挂载到根module上的state中，所以我们可以这样访问模块数据
```
const store = new Vuex.Store({
  modules: {
    a: {
      state: {
        count: 1
      }
    }
  }
  state: {
    count: 0
  }
})

this.$store.state.a.count // 1
```

而在模块内部访问数据时，我们又不必依照上面这种格式，这是因为我们在本地化的时候，针对`mutations`和`actions`会传入对应模块的state作为参数
```
Object.defineProperties(local, {
  state: {
    get: () => getNestedState(store.state, path)
  }
})
```

### Getter

在组件中我们通过`this.$store.getters`来拿到getter方法，这是因为我们在`resetStoreVM`的时候，把`store.getter`的方法访问指向`store._vm[key]`，能通过`store._vm[key]`访问到实际getter方法是因为我们把`store._wrappedGetters`中的方法作为computed传入Vue生成计算属性
```
computed[key] = partial(fn, store)
Object.defineProperty(store.getters, key, {
  get: () => store._vm[key],
  enumerable: true // for local getters
})

store._vm = new Vue({
  data: {
    $$state: state
  },
  computed
})
```

在模块内部时，针对`mutations`和`actions`会传入对应模块的getter作为参数，本地化的工作是通过`makeLocalGetters`方法完成的，我们遍历`store.getters`然后截取掉命名空间，接着把截取后的方法代理到真实的getters方法中，这样在内部模块访问时就能找到正确的getter方法

## 修改数据

修改数据最终都是通过`commit`执行`mutation`方法，我们在`action`中异步操作完成后也是执行commit

Store类定义了`commit`和`dispatch`两个方法，分别用来执行_mutations和_actions中的方法

`commit`的实现是首先通过传入的`type`从`_mutations`找到对应的方法，然后先通过`_withCommit`解除锁定，再修改state，这样通过`commit`的修改就不会引发报错

`dispatch`的实现类似，不同是根据`type`从`_actions`找到对应的方法数组后，会先转成Promise，然后执行`Promise.all`得到最终结果并返回一个Promise

至于模块内部的修改，我们在本地化时会重载`commit`和`dispatch`方法，所以在提交方法的时候，其实会先拼接命名空间找到真正的方法然后再执行原始的`commit`和`dispatch`方法

## 语法糖

Vuex提供了`mapState`、`mapGetters`、`mapMutations`、`mapActions`四个语法糖，简化我们操作Vuex的流程并降低心智负担

这四个语法糖的实现大同小异，首先通过`normalizeNamespace`格式化传入的参数，第一个为命名空间，第二个为数组或对象，如果不传第二个参数，则把第一个参数当成第二个参数传入，命名空间置为空；接着把用户传入的数组和对象规范成统一结构方便处理，接着我们通过`this.$store`拿到全局的操作方法，如果存在命名空间，则根据命名空间拿到具体的模块并从其`context`属性拿到各模块对应的本地方法，最后返回包含真正操作方法的结果对象

规范用户输入成如下格式：

```
normalizeMap(['count']) => [ { key: 'count', val: 'count' } ]
normalizeMap({a: 1, b: 2, c: 3}) => [ { key: 'a', val: 1 }, { key: 'b', val: 2 }, { key: 'c', val: 3 } ]
```

具体使用方法：
```
var vm = new Vue({
  el: '#app',
  computed: {
    ...mapState(['count']),
    ...mapState('a', {
      aCount: 'count'
    }),
    ...mapGetters(['computedCount']),
    ...mapGetters('b', {
      bComputedCount: 'computedCount'
    })
  },
  methods: {
    ...mapMutations(['increment']),
    ...mapMutations('a', {
      aIncrement: 'increment'
    }),
    ...mapActions({
      actIncrement: 'increment'
    }),
    ...mapActions('b', {
      bActIncrement: 'increment'
    }),
    add () {
      const { commit } = this.$store
      commit('increment')
    }
  },
  store
})
```