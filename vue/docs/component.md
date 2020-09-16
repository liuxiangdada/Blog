# 组件

## 组件注册

组件注册有两种方式，一种是通过`Vue.component`方法，另一种是在组件定义中添加`components`选项

- 前者接收组件名和配置对象或者函数（后面提到），然后将对象转成组件构造函数注入到Vue的options的components属性中，这个属性其实在初始化时已经被注入了三个内置组件，这样，我们在其他组件创建时就会合并Vue.options也就自然继承了这些组件
- 后者是根据渲染节点时引用的组件名，然后从该组件实例的components选项中提取出配置对象，然后在创建组件渲染VNode时将其转成构造函数供组件实例的创建，这种注入方法因为只是在创建组件构造函数时把自身添加到组件构造函数options的componetns属性中，并没有提升到其原型内，因此只在该组件内可用

## 组件创建

new Vue 时，对待普通节点（通过传入el、template、render函数生成）和对待组件的patch过程是不一样的

new Vue时会根据挂载的el，或者手动指定的id找到真实DOM，并依据该DOM创建一个占位符VNode，然后把该占位符VNode作为$vnode，在组件挂载的时候生成一个渲染VNode作为_vnode，当组件渲染到一个嵌套组件时，会先根据该子组件创建一个占位符VNode，接着在该子组件挂载的时候生成新的渲染VNode，以此渲染出整个组件树

在递归渲染的过程中，通过闭包保存每次渲染的父实例，以此调用父组件的数据方法，子组件patch完成后，恢复父组件的渲染

vue-loader将template转化为render函数时，将其中所有的节点都执行createElement方法创建VNode节点，遇到组件创建占位符节点，这样我们有几个组件，就会得到几个VNode列表

渲染的时候，真实DOM的插入是递归的，我们拿到一个VNode节点，根据其tag创建一个真实标签，然后遍历其children数组，将子节点插入到该标签中，如果子节点包含子节点则递归插入，所以DOM的插入顺序是先子后父的，只有在所有的节点都插入完成后，外层节点才会被插入到body中

Vue带编译器版本可以将html解析成JSON串，包含了每个节点的各种信息，如标签名、类名、样式、属性、绑定事件等，然后将这些信息传入内置的createElement函数，所以其实不论我们手写render函数、vue编译器编译、vue-loader解析vue文件的template最终都是拿到这些信息，经过createElement处理后得到VNode，交给patch处理插入到真实DOM中，值得注意的是，对于HTML中的组件而言，与其他节点并没有什么不同，解析到组件节点时，会去实例的$options属性中查找是否在components中导入该组件的配置，继而生成该组件的VNode

我们以Vue官方Hello World Demo的渲染流程来梳理一下
- 首先new Vue，生成Vue实例，由于指定了render函数，且传入的是组件配置类App，所以在h函数中进入创建组件VNode逻辑
- patch的时候，发现这个VNode是组件节点，创建组件App实例，进入组件的渲染过程
- 由于vue-loader会将template中的html转化成render函数，所以生成的App组件渲染VNode为_vnode，上面创建的占位符VNode为$vnode
- 进行App组件的渲染，期间创建子节点时发现了HelloWorld子组件，创建该子组件实例，同样的把该子组件占位符VNode设为$vnode，得到子组件的渲染VNode为_vnode
- HelloWorld子组件的渲染VNode渲染期间再没有嵌套组件，走普通节点的创建流程，patch结束后，将真实DOM返回到HelloWorld组件实例的$el上，同时也映射到其占位符节点的componentInstance上，然后我们在占位符VNode的elm属性得到真实的DOM，插入App组件实例的DOM节点中
- 同理，App组件渲染完毕，插入body中

## 组件生命周期

由于DOM节点的插入是先子节点后父节点，因此，组件间的生命周期是：父created -> 子created -> 子mounted -> 父mounted
销毁组件时，先执行beforeDestory钩子，然后递归删除子组件，所以先执行子组件的destory钩子

## 异步组件

如果你定义了全局的异步加载组件，然后又在某个组件内再次定义该组件，那么在初始化过程中，即使异步加载完成也不会再次进入resolveAsyncComponent方法，因为此时子组件的构造器已拿到

异步组件有三种加载方式，分别为工厂函数、Promise、高级异步组件

工厂函数模式下，合入components属性中的既不是对象也不是组件构造函数，而是一个匿名工厂函数，所以进入resolveAsyncComponent方法，里面定义了异步组件的resolve、reject、forceRender三个方法，第一次执行时，返回undefined，创建一个注释节点，等到异步加载完毕后，调用resolve方法，取得加载到的组件构造函数，再调用forceRender强制当前激活实例重新渲染，这样就渲染出异步组件了

高级异步组件的情况下，我们传入的函数会返回一个配置对象，在factory函数执行后得到的res就是我们定义的配置，这时检查有无component，有则异步加载组件，并且保存loading组件和error组件的构造器，这时如果没有定义delay，则返回loading组件的构造器，渲染出loading组件，否则返回undefined直到delay定时器结束后，返回loading组件构造器并渲染出loading组件，如果设置了timeout，则会在超时后返回error组件的构造器并渲染出error组件

异步组件的本质是两次以上的渲染，首先渲染成注释节点，再通过forceRender重新渲染