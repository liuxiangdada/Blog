# 扩展功能

## 事件

Vue对DOM事件和自定义事件两类分别处理，主要区别在于绑定事件和移除事件的不同

### DOM事件

DOM事件通过解析模板上绑定的属性，提取出修饰符，提前插入修饰符对应的代码，返回一个包装函数，这样在调用时就能实现不同的功能（阻止冒泡、阻止默认事件等）

得到事件的封装JSON串后，我们就可以在渲染DOM的时候进行绑定，绑定发生在DOM节点的创建(_createElm)中，即不论我们创建子节点还是创建组件节点都会在完成后调用created钩子，该钩子调用updateDOMListeners方法，其内部又实际调用updateListeners方法对新事件进行包装以便支持对同一事件绑定多个回调，最终调用add方法将其绑定到真实DOM上

更新事件和绑定事件调用同一updateListeners方法，由于封装事件invoke其实是apply其fns属性，所以当我们修改回调后，只需要将新的回调赋值给老invoke的fns属性并将老的invoke赋值给新vnode的data的on属性即可

### 组件原生事件

组件上可以绑定原生事件，在编译时会判断native修饰符并将其加入到ast节点的nativeOn属性中保存，不同于on属性，在组件vnode创建时，会把on属性赋值给listeners走自定义事件逻辑，而将nativeOn属性赋值给data.on走DOM事件逻辑，因为DOM节点的挂载是先子后父，所以等到子组件拿到elm真实节点后才会进行事件的绑定（发生在initComponent方法调用的created钩子中），所以这个时候原生事件会直接绑定在子组件的根节点上
```
vnode.elm = vnode.componentInstance.$el
if (isPatchable(vnode)) {
  invokeCreateHooks(vnode, insertedVnodeQueue)
  setScope(vnode)
}
```

这里调用`invokeCreateHooks`方法就会去平台的`event.js`文件中找到事件的create钩子函数，执行`updateDOMListeners`方法绑定事件

### 自定义事件

自定义事件在initEvent中进行初始化，但其事件回调是在编译时得到的组件ast节点的on属性，经由createComponent创建组件vnode时把on属性赋值给listeners，然后在组件的init合并组件配置过程传入到_parentListeners属性，最后在initEvent方法中调用updateListeners方法，走和DOM事件绑定相似的逻辑，区别是封装好的invoke回调不是绑定到DOM上，而是作为实例_event对象的key-value存放，等到$emit方法调用时，从_event中取出该回调执行，所以我们知道了其实组件emit某个方法并不是回到父组件去执行该回调，而是访问自身实例的_event属性中有无该方法并执行，其执行环境仍然在子组件中，这样我们就可以通过传值把子组件中的数据传入到回调在父组件中取得子组件数据，这样，我们就可以实现父子通信

**为什么把事件从父组件传递到子组件中保存后，执行时函数中的this仍指向父组件实例**
前面我们讨论了子组件能够通过emit触发父组件的方法，并能把子组件数据作为参数传递到父组件，这样我们就能在父组件的方法中使用子组件的数据，但我没有说明在该回调中为何能够使用父组件的数据，我们知道emit的实质是在当前实例的_event中寻找对应名称的回调，_event属性又是在父组件渲染时传递而来，按照常理，虽然该方法是父组件定义的方法，但其执行环境已经位于子组件，this应该指向子组件，但因为我们在初始化initMethods的方法中，将写在methods属性中的方法修改到实例上并用bind硬绑定this为当前实例（父实例），所以即使是在子组件中调用因其原函数已经被绑定为Vue实例，故而可以拿到父组件的数据

## 双向绑定（v-model）

Vue的响应式对象其实并没有实现双向绑定，只完成了从数据侧到视图层的响应，而视图层的变化是没有响应到数据侧的，但Vue提供双向绑定的能力，通过v-model实现

### 表单元素的v-model

v-model作用于表单元素实质是语法糖，ast节点在generate时，会new CodegenState，这个类干了一件事就是把baseOptions中定义的全局directives（html、text、model）合并到自定义配置中作为编译参数，接下来在genData时发现是指令则会调用genDirectives方法解析，接着去找options中的model指令，执行定义的model函数，因为是普通文本input，命中genDefaultModel方法，该方法主要处理了下面几件事：
- 针对绑定了v-bind的属性，不能再绑定v-model
- 获取v-model上的修饰符，注意lazy修饰符，其本质是改为监听change事件而不是input事件
- 生成赋值语句，把input事件的value赋值给绑定prop并提供composing判定，当composing为true时，直接返回（后面讲作用）
- 最后给该ast节点增加名为value的prop和input事件

上面针对v-model指令的编译阶段的行为作出了分析，在运行时又是如何触发的呢？

会在节点创建时触发vnode的create钩子，调用updateDirectives方法，该方法调用_update，由于创建时没有oldVnode，所以我们把注入到Vue.options中的指令（show、model）的inserted钩子和componentUpdated钩子合并到vnode的insert钩子中，这样在patch最后执行invokeInsertHook方法时就会执行inserted钩子，该钩子就绑定了`compositionstart`事件和`compositionend`事件

我们在input上输入中文结束后就会触发compositionend事件进而内部trigger触发input事件执行回调进行赋值，虽然这时更新页面同时也触发了update钩子，会再次进入updateDirectives方法，但这时由于`dirsWithInsert`数组为空，只会触发model的componentUpdated钩子，而componentUpdated钩子是针对select的，所以不会有其他变化
 ```
const dirsWithInsert = []
const dirsWithPostpatch = []

let key, oldDir, dir
for (key in newDirs) {
  oldDir = oldDirs[key]
  dir = newDirs[key]
  // 触发update钩子时，新旧指令都存在，只往dirsWithPostpatch中添加dir
  if (!oldDir) {
    // new directive, bind
    callHook(dir, 'bind', vnode, oldVnode)
    if (dir.def && dir.def.inserted) {
      dirsWithInsert.push(dir)
    }
  } else {
    // existing directive, update
    dir.oldValue = oldDir.value
    dir.oldArg = oldDir.arg
    callHook(dir, 'update', vnode, oldVnode)
    if (dir.def && dir.def.componentUpdated) {
      dirsWithPostpatch.push(dir)
    }
  }
}

if (dirsWithInsert.length) {
  const callInsert = () => {
    for (let i = 0; i < dirsWithInsert.length; i++) {
      callHook(dirsWithInsert[i], 'inserted', vnode, oldVnode)
    }
  }
  if (isCreate) {
    mergeVNodeHook(vnode, 'insert', callInsert)
  } else {
    callInsert()
  }
}

if (dirsWithPostpatch.length) {
  mergeVNodeHook(vnode, 'postpatch', () => {
    for (let i = 0; i < dirsWithPostpatch.length; i++) {
      callHook(dirsWithPostpatch[i], 'componentUpdated', vnode, oldVnode)
    }
  })
}
 ```

上面的分析可以看出v-model指令真正的实现就是添加了一个value作为prop和input事件，写法等同于：
```
<input :value="message" @input="if($event.target.composing)return;message=$event.target.value">
```

所以我们自己实现上面的代码和v-model是一样的，区别在于`if($event.target.composing)return;`这一句，一般而言，我们修改input值会立即触发input事件修改prop，但我们输入中文时会有一个确认的过程，这时其实并不希望触发input方法，Vue正是帮我们做了这件事

前面提到在inserted钩子中监听了`compositionstart`事件和`compositionend`事件，在输入合成开始时将composing置为true，这样就无法进入input事件的赋值逻辑，等到输入合成结束触发compositionend事件再手动触发input事件进行赋值

### 组件的v-model

组件的v-model在编译阶段和表单元素不同在执行model函数时，命中genComponentModel方法，该方法为ast节点添加model属性记录model的回调函数、表达式、真实值三个属性，使用的地方在创建组件vnode时，会判断data中有无model属性并进入transformModel方法，该方法接收组件options中的model选项，使得我们可以设置props和event，如果不设置，默认为value和input事件，接着为data添加attrs属性保存prop的值和名称；添加on属性保存指定的事件名和data.model中保存的回调，用法如下：
```
let Child = {
  template: '<div>' +
  '<input :value="msg" @input="updateValue" placeholder="edit me">' +
  '</div>',
  model: {
    prop: 'msg',
    event: 'change'
  },
  props: ['msg'],
  methods: {
    updateValue(e) {
      this.$emit('change', e.target.value)
    }
  },
}
```

可以看到，我们对组价使用`v-model`时，还需要手动调用`emit`触发我们设置的`change`事件，change事件内部其实和v-model一样根据修饰符封装好一个函数，内部修改我们绑定的prop的值

## 插槽

### 老语法（2.6之前）

#### 具名插槽

在编译阶段，父组件中将子组件内部的chilren作为插槽vnode，在每个子节点上添加attrs属性记录形如`attrs: { slot: 'header' }`以及使用slot属性记录插槽名称，形如`slot: 'header'`

子组件编译阶段处理`slot`标签时，会记录`slotName`并在genSlot中将slotName传入`_t -> renderSlots`方法中，最后得到形如`_t('header', children)`的code，children指代递归生成的插槽默认的子vnode节点

接下来在子组件初始化initRender方法中，会取得子组件的占位符vnode中的children，以slot名称为键赋值给实例的`$slots`属性，形如`vm.$slots = { header: [VNode] }`，然后在renderSlot方法中，将$slot属性中对应slotName的子节点返回到子组件的渲染函数中渲染出来

#### 作用域插槽

编译parse阶段，父组件解析ast节点到template标签时，会取得其slot-scope属性保存在自身ast节点的`slotScope`属性上，并在子组件占位符节点的`scopedSlots`属性上添加该template节点，在generate阶段的genData方法中，发现存在scopedSlots属性进入genScopedSlots方法，根据slot名称遍历slotScope属性，对每个template节点都拿到其slot-scope属性指定的参数props，然后构造一个返回template节点子节点的函数，然后把函数和key以对象数组的形式传入resolveScopedSlots方法中处理，并作为子组件的data属性返回，generate后生成`scopedSlots: _u([{ key: 'header', fn: function (props) { return _c('p', [_v(_s(props.text))]) } }])`的code

子组件在编译时处理`slot`标签时，跟具名插槽不一样的是会从节点的attrs属性中拿到子组件中传递的数据作为props，`_t("default",null,{"text":"Hello","msg":msg})`代码中第三个参数即为props

和具名插槽有点不一样，这时子组件占位符vnode没有children，所以`$slot`属性为空，但是后面在_render时，通过normalizeScopedSlots方法可以取得上面生成的子组件data中的scopedSlots属性，将插槽名对应的函数赋值到实例的`$scopedSlots`属性中

这样在进入renderSlot渲染时，就能根据`$scopedSlots`对象和插槽名拿到对应的scopedSlotFn，将props作为参数调用scopedSlotFn，在子组件渲染时延时得到父组件中设置在template节点中的vnode子节点，这样就能在父组件中访问子组件的数据

### 新语法

自2.6版本起，使用`v-slot`代替了混乱的`slot`和`slot-scope`，新语法如下：
```
// 具名插槽
<template v-slot></template>
<template v-slot:header></template>
<template #header></template>

// 作用域插槽
<template v-slot:header="props"></template>
<template v-slot:header="{ user }">
  <p>{{ user.firstName }}</p>
</template>
<child v-slot:default="props"></child>
<child #default="props"></child>
```

#### 具名插槽

新版本父组件在编译阶段大致和原来相同，不同在于parse阶段会把template标签的子节点赋值到父ast节点的scopedSlots属性上，不包裹在template中的节点才作为子组件占位符节点的children；碰到直接写在子组件节点上的`v-slot`，会判断是否是在组件上使用；是否使用了老语法；是否是默认插槽；不满足报错处理，如果符合要求，那么会创建一个template节点把该子组件下的children移到template节点下，清空children，最后赋值给scopedSlots属性，使得好像是用户手写了一个`<template v-slot:default></template>`一样，而子组件中没有被template包裹的节点统统作为子组件的children在创建子组件vnode时作为componentOptions传入最终作为`$slot`属性，进而通过normalizeScopedSlots方法被规范到`$scopedSlots`属性中，被子组件渲染时拿到

#### 作用域插槽

新语法里，作用域插槽和具名插槽的区别是在编译阶段，作用域插槽就被封装成一个函数，等到子组件渲染时才被转化成vnode，而具名插槽在父组件阶段就已经被转成vnode，不过后来将`$slot`属性规范为`$scopedSlots`属性的函数引用而已，所以新语法总的来说两种插槽的写法处理趋于一致，沿袭了原来的需求，即作用域插槽延时渲染

## keep-alive

keep-alive组件的本质是缓存实例化的组件vnode，下次触发渲染时命中缓存就直接从cache中取得vnode，而切换后的组件不再需要经历render、mounted的过程

keep-alive组件使用时要注意
- 只能作用于组件，其内部每次只能渲染一个组件，并且内部不能包含除组件外的其他节点
- 内部的组件必须设置name属性，不然无法成功取得对应组件（其实这时会取组件的tag）
- keep-alive的本质是拿空间换时间的一种实现，如果缓存组件过多，性能会下降，最好设置max作为限制，这样在超出max范围后就会删除最久没被使用的组件vnode

### keep-alive的实现

keep-alive组件是内置组件，在initGlobalAPI方法中作为全局组件被注册，所以在任何地方都能使用

其实现在`core/components/keep-alive`文件中，内部主要是手动定义了keep-alive组件的配置对象，重点是手写了render函数，该函数返回内部第一个组件的vnode并将其缓存到cache中

值得一提的是render函数中取内部组件的方式和`slot`是类似的，所以可理解为keep-alive是另类的slot使用

keep-alive组件还定义三个props：`include/exclude/max`用于配置哪些组件是否需要被缓存以及最大缓存组件数，在mounted钩子中对`include/exclude`进行监听，把不需要缓存的vnode销毁并从cache中移除，在移除时利用LRU思想做了优化，每次命中缓存就会把命中的vnode的key放到keys数组的末尾，这样在移除时移除的第一个元素就是最久没被使用的vnode

### keep-alive组件的初次渲染

初次渲染走到patch，然后渲染到keep-alive组件时触发keep-alive组件的实例化并进入keep-alive组件的渲染逻辑，这时就进入keep-alive组件的render函数，我们取到keep-alive组件实例`$slots`属性的default即代表拿到其内部的子节点，这时再取出第一个组件节点，加入缓存并返回该组件vnode

渲染keep-alive组件最终得到的是其第一个组件的vnode，所以又会进入组件vnode的渲染，组件渲染完成后接着渲染keep-keep-alive组件，执行其mounted钩子，至此，keep-alive组件初次渲染完毕

### keep-alive组件的再次渲染

切换keep-alive组件内的动态组件时，会进入页面的patch流程，在patchVnode触发prepatch钩子时，就会执行updateChildComponent方法，由于这时新旧vnode为不同组件的vnode，所以会触发resolveSlots方法修改keep-alive组件的`$slots`属性并重新渲染，这样进入keep-alive组件的重新渲染，如果命中缓存就会在缓存中拿到该组件的vnode，然后进入新组件的渲染流程，根据vnode创建组件的过程中调用`createComponent`方法，这时由于前面已经标记了keepAlive属性，在进入组件的init钩子时就不会再去创建组件实例并mount而是直接执行prepatch钩子，重点是这时的`isReactivated`为true，会命中reactivateComponent方法将新组件插入到DOM中，回到patch函数中，由于新旧节点不是相同类型的节点，所以会删除旧组件，接着会触发旧组件的deactivated钩子，最后在flushSchedulerQueue方法执行完watcher的run方法后，统一调用callActivatedHooks触发新组件的activated钩子，处于keep-alive组件中的组件在切换时不会再触发created、mounted等钩子

## transition

transition组件同样是Vue的内置组件，被用来实现节点出现或消失时的过渡效果，可指定过渡期间不同阶段的类名实现过渡效果，也可以使用JS钩子调用动画库实现高级动画

当Vue渲染到transition组件节点时，由于transition组件自定义了render函数，所以执行在`src/platforms/components/transition.js`中自定义的render函数逻辑，根据`this.$slots.default`拿到子节点，接着进行校验，诸如不能多个节点、模式只能是in-out或者out-in等情况进行报错，然后拿到第一个子节点，给子节点设置以`__transition-`为前缀的key，接着赋值子节点`data.transition`属性，值是设置在transition上的props（包含一些控制参数如appear、mode，还有一些过渡类名等），最后把该子节点返回作为vnode渲染

在创建patch函数时，我们提到会把不同平台的钩子注入进来，所以已经把transition的_enter函数注入到节点patch的create钩子，把remove注入到节点patch的destory钩子中

等到子节点渲染时，在创建后就会触发created钩子，执行_enter方法，该方法首先会检查是否有`data.transition`属性，这样就能过滤掉非transition组件内部的节点，校验通过后提取transition设置的一系列props，接着我们判断`isAppear`是否为真，并且针对是否设置`appear`prop决定要不要首次就进行过渡，然后我们根据props得到enter过渡的一系列类名、钩子函数

这个时候我们添加`enter`和`enter-active`类名，等到nextFrame（浏览器的逐帧刷新机制）回调触发时，首先删除`enter`类名，然后添加`enter-to`类名，接着执行whenTransitionEnds方法，该方法会绑定一个监听函数监听`transitionend`事件并在duration到期后执行定时器回调`end`，内部首先移除监听，然后调用回调cb，cb是我们在前面定义的`_enterCb`方法，该方法，会移除`enter-to`和`enter-active`类

节点在移除时会执行leave钩子，流程和_enter钩子是类似的，只不过类名不一样

至此，我们完成了css节点在各个时间节点的插入移除过程，保证了过渡的实现，同时在过渡的各个节点执行我们定义的JS钩子函数（如果有的话）

编译阶段：transition组件在数据变更引发内部节点消失时，会在父组件的渲染时把该移除的vnode移除掉（生成render函数时针对v-if指令采取三元运算符），接着进入updateChildComponent方法更新`$slots`，因为移除后会填补一个空白的文本节点，所以在`resolveSlots`被过滤掉，所以下次进入transition的render函数时，已经拿不到子节点了

## transition-group

transition-group在自定义render中首先拿到props中的tag，如果没有设为span，然后创建一个map保存子节点的key和vnode（所以子节点必须指定key），接着取出子节点，遍历把设置在transition-group上的props作为子节点的`data.transtition`属性，和上面一样，最后根据tag和children创建出一个vnode返回进行渲染

继续渲染，走到vm._update方法时，由于在transition-group组件的beforeMounted钩子中重写了该方法，所以会执行该重写方法，此时当成正常的_update方法即可，这样就渲染出了transition-group中的节点

在重写的_update方法中执行了两次patch，官方的说法是因为updateChild的算法不稳定，于是在第一次时删除多余的节点，并触发leave钩子，在第二次时，保证留下的节点在它正确的位置

### 增加节点

接着我们添加一个元素触发enter逻辑，就会再次进入render函数，这时我们首先保存上次的子节点作为`prevChildren`，注意这时的子节点是比prevChildren多一个的，我们首先对prevChildren进行处理，保存每个节点当前的位置信息到`data.pos`，然后根据tag和children创建出一个vnode返回进行渲染

继续渲染触发_update，增加节点时也走正常update逻辑，由于是新增节点，所以在第一次update时因为比较_vnode和kept（kept下面提）没有什么变化，等到第二次update时，vnode是比_vnode多一个节点的，所以在patchVnode时会在对应位置插入节点，我们在上面transition时提到有_enter钩子方法和remove钩子方法，因为我们前面为每个子节点都添加了`data.transition`属性，所以在插入时就会自动添加`enter`和`enter-active`类

这时我们来到transition-group组件的updated生命周期，一开始会检测节点是否定义了moveclass并且class实现和过渡动画相关，如果不满足啥也不干，否则会对子节点执行以下三步
- 如果我们快速添加节点，则会执行callPendingCbs方法，立即执行moveCb回调和enerCb回调
- 记录添加后子节点的位置记为`data.newPos`
- 比较前后两次位置，如果有差异，则把子节点恢复到原来的位置

接着我们调用`document.body.offsetHeight`执行一次浏览器重绘，这时我们看到DOM上所有节点因为被新节点撑开的空间又消失了，接着我们遍历每个子节点，添加`move-class`，并把移动的位置复原，这样原来的节点就会有被撑开的过渡，使得看起来不那么生硬，最后和上面transition一样，完成进入后的过渡

### 删除节点

删除节点重新进入render函数，有一点不同，就是我们在比较map时，会把仍然在的接待保存到`this.kept`，把删除的节点保存到`this.removed`，这样在执行重写的_update方法时，就会在第一次patch在updateChildren时最后删除多余的节点，就会触发leave钩子，在里面添加`leave`和`leave-active`类，第二次update时就已经一样了，接着又走到updated周期钩子，这时和增加节点一样，会先回复原位，然后在回调中复原，最后完成离开的过渡并在_leaveCb方法中移除那个节点
