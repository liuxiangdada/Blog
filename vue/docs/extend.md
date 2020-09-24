# 扩展功能

## 事件

Vue对DOM事件和自定义事件两类分别处理，主要区别在于绑定事件和移除事件的不同

### DOM事件

DOM事件通过解析模板上绑定的属性，提取出修饰符，提前插入修饰符对应的代码，返回一个包装函数，这样在调用时就能实现不同的功能（阻止冒泡、阻止默认事件等）

得到事件的封装JSON串后，我们就可以在渲染DOM的时候进行绑定，绑定发生在DOM节点的创建(_createElm)中，即不论我们创建子节点还是创建组件节点都会在完成后调用created钩子，该钩子调用updateDOMListeners方法，其内部又实际调用updateListeners方法对新事件进行包装以便支持对同一事件绑定多个回调，最终调用add方法将其绑定到真实DOM上

更新事件和绑定事件调用同一updateListeners方法，由于封装事件invoke其实是apply其fns属性，所以当我们修改回调后，只需要将新的回调赋值给老invoke的fns属性并将老的invoke赋值给新vnode的data的on属性即可

### 组件原生事件

组件上可以绑定原生事件，在编译时会判断native修饰符并将其加入到ast节点的nativeOn属性中保存，不同于on属性，在组件vnode创建时，会把on属性赋值给listeners走自定义事件逻辑，而将nativeOn属性赋值给data.on走DOM事件逻辑，因为DOM节点的挂载是先子后父，所以等到子组件拿到elm真实节点后才会进行事件的绑定（发生在initComponent方法调用的created钩子中），所以这个时候原生事件会直接绑定在子组件的根节点上

### 自定义事件

自定义事件在initEvent中进行初始化，但其事件回调是在编译时得到的组件ast节点的on属性，经由createComponent创建组件vnode时把on属性赋值给listeners，然后在组件的init合并组件配置过程传入到_parentListeners属性，最后在initEvent方法中调用updateListeners方法，走和DOM事件绑定相似的逻辑，区别是封装好的invoke回调不是绑定到DOM上，而是作为实例_event对象的key-value存放，等到$emit方法调用时，从_event中取出该回调执行，所以我们知道了其实组件emit某个方法并不是回到父组件去执行该回调，而是访问自身实例的_event属性中有无该方法并执行，其执行环境仍然在子组件中，这样我们就可以通过传值把子组件中的数据传入到回调在父组件中取得子组件数据，这样，我们就可以实现父子通信

**为什么把事件从父组件传递到子组件中保存后，执行时函数中的this仍指向父组件实例**
前面我们讨论了子组件能够通过emit触发父组件的方法，并能把子组件数据作为参数传递到父组件，这样我们就能在父组件的方法中使用子组件的数据，但我没有说明在该回调中为何能够使用父组件的数据，我们知道emit的实质是在当前实例的_event中寻找对应名称的回调，_event属性又是在父组件渲染时传递而来，按照常理，虽然该方法是父组件定义的方法，但其执行环境已经位于子组件，this应该指向子组件，但我们在初始化的initMethods的方法中，将写在methods属性中的方法修改到实例上并用bind硬绑定this为当前实例，所以即使是在子组件中调用因其原函数已经被绑定为Vue实例，故而可以拿到父组件的数据

## 双向绑定（v-model）

Vue的响应式对象其实并没有实现双向绑定，只完成了从数据侧到视图层的响应，而视图层的变化是没有响应到数据侧的，但Vue提供双向绑定的能力，通过v-model实现

### 表单元素的v-model

v-model作用于表单元素实质是语法糖，ast节点在generate时，会new CodegenState，这个类干了一件事就是把baseOptions中定义的全局directives（html、text、model）合并到自定义配置中作为编译参数，接下来在genData时发现是指令则会调用genDirectives方法解析，接着去找options中的model指令，执行定义的model函数，因为是普通文本input，命中genDefaultModel方法，该方法主要处理了下面几件事：
- 针对绑定了v-bind的属性，不能再绑定v-model
- 获取v-model上的修饰符，注意lazy修饰符，其本质是改为监听change事件而不是input事件
- 生成赋值语句，把input事件的value赋值给绑定prop并提供composing判定（后面讲作用）
- 最后给该ast节点增加名为value的prop和input事件

上面针对v-model指令的编译阶段的行为作出了分析，在运行时又是如何触发的呢？

会在节点创建时触发vnode的create钩子，调用updateDirectives方法，该方法调用_update，由于创建时没有oldVnode，所以我们把注入到Vue.options中的指令（show、model）的inserted钩子和componentUpdated钩子合并到vnode的insert钩子中，这样在patch最后执行invokeInsertHook方法时就会执行inserted钩子，该钩子就绑定了`compositionstart`事件和`compositionend`事件

我们在input上输入中文时就会触发compositionend事件进而内部trigger触发input事件执行回调进行赋值，虽然这时更新页面同时也触发了update钩子，会再次进入updateDirectives方法，这时只会触发model的componentUpdated钩子，而componentUpdated钩子是针对select的，所以不会有其他变化

上面可以看出v-model指令真正的实现就是添加了一个value作为prop和input事件，写法等同于：
```
<input :value="message" @input="if($event.target.composing)return;message=$event.target.value">
```

所以我们自己实现上面的代码和v-model是一样的，区别在于`if($event.target.composing)return;`这一句，一般而言，我们修改input值会立即出发input事件修改prop，但我们输入中文时会有一个确认的过程，这时其实并不希望出发input方法，Vue正是帮我们做了这件事

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

## 插槽

### 老语法（2.6之前）

#### 具名插槽

在编译阶段，父组件中将子组件内部的chilren作为插槽vnode，在每个子节点上添加attrs属性记录形如`attrs: { slot: 'header' }`以及使用slot属性记录插槽名称，形如`slot: 'header'`

子组件编译阶段处理`slot`标签时，会记录slotName并在genSlot中将slotName传入`_t()`方法中，最后得到形如`_t('header', children)`的code，children指代递归生成的子vnode节点

接下来在子组件初始化化initRender方法中，会取得父组件中子组件的占位符vnode中的children，将其以slot名称为键赋值给实例的`$slots`属性，形如`vm.$slots = { header: [VNode] }`，然后在renderSlot方法中，将$slot属性中对应slotName的子节点返回到子组件的渲染函数中渲染出来

#### 作用域插槽

编译parse阶段，父组件解析ast节点到template标签时，会取得其slot-scope属性保存在子组件占位符ast节点的`slotScope`属性上，并将自身赋值给子组件占位符节点的scopedSlots属性，在generate阶段的genData方法中，发现scopedSlots属性进入genScopedSlots方法，根据slot名称遍历slotScope属性，对每个template节点都拿到其slot-scope属性指定的参数props，然后构造一个返回template节点子节点的函数，最终返回形如`scopedSlots: _u([{ key: 'header', fn: function (props) { return _c('p', [_v(_s(props.text))]) } }])`的code

子组件在编译时处理`slot`标签时，跟具名插槽不一样的是会从节点的attrs属性中拿到子组件中传递的数据作为props，形如`_t("default",null,{"text":"Hello","msg":msg})`

和具名插槽有点不一样，这时父组件中的子组件占位符vnode没有children，所以`$slot`属性为空，但是后面在_render时，通过normalizeScopedSlots方法可以取得上面生成的子组件data中的scopedSlots属性，将插槽名对应的函数赋值给实例的`$scopedSlots`属性

这样在进入renderSlot渲染时，就能根据`$scopedSlots`对象和插槽名拿到对应的scopedSlotFn，将props作为参数调用scopedSlotFn，在子组件渲染时延时得到父组件中设置在template节点中的子节点，这样就能在父组件中访问子组件的数据

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

新版本父组件在编译阶段大致和原来相同，不同在于parse阶段会把template子节点赋值给父ast节点的scopedSlots属性上，而不是作为子组件占位符节点的children；碰到直接写在子组件节点上的`v-slot`，会判断是否是在组件上使用；是否使用了老语法；是否是默认插槽；不满足报错处理，如果符合要求，那么会创建一个template节点把该子组件下的children移到template节点下，清空children，最后赋值给scopedSlots属性，使得好像是用户手写了一个`<template v-slot:default></template>`一样，而子组件中没有被template包裹的节点统统作为子组件的children在创建子组件vnode时作为componentOptions传入最终作为`$slot`属性，进而通过normalizeScopedSlots方法被规范到`$scopedSlots`属性中，被子组件渲染时拿到

#### 作用域插槽

新语法里，作用域插槽和具名插槽的区别是在编译阶段，作用域插槽就被封装成一个函数，等到子组件渲染时才被转化成vnode，而具名插槽在父组件阶段就已经被转成vnode，不过后来将`$slot`属性规范为`$scopedSlots`属性的函数引用而已，所以新语法总的来说两种插槽的写法处理趋于一致，沿袭了原来的需求，即作用域插槽延时渲染

## keep-alive

## transition