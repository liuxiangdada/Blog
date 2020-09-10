# 更新视图

## Vnode

## 虚拟DOM

## 更新策略

new Vue 时，对待普通节点（通过传入el、template、render函数生成）和对待组件的patch过程是不一样的

new Vue时会根据挂载的el，或者手动指定的id找到真实DOM，并依据该DOM创建一个占位符VNode，然后把该占位符VNode作为$vnode，在组件挂载的时候生成一个渲染VNode作为_vnode，当组件渲染到一个嵌套组件时，会先根据该子组件创建一个占位符VNode，接着在该子组件挂载的时候生成新的渲染VNode，以此渲染出整个组件树

在递归渲染的过程中，通过闭包保存每次渲染的父实例，以此调用父组件的数据方法，子组件patch完成后，恢复父组件的渲染

vue-loader将template转化为render函数时，将其中所有的节点都执行createElement方法创建VNode节点，遇到组件创建占位符节点，这样我们有几个组件，就会得到几个VNode列表

渲染的时候，真实DOM的插入是递归的，我们拿到一个VNode节点，根据其tag创建一个真实标签，然后遍历其children数组，将子节点插入到该标签中，如果子节点包含子节点则递归插入，所以DOM的插入顺序是先子后父的，只有在所有的节点都插入完成后，外层节点才会被插入到body中



我们以Vue官方Hello World Demo的渲染流程来梳理一下
- 首先new Vue，生成Vue实例，由于指定了render函数，且传入的是组件配置类App，所以在h函数中进入创建组件VNode逻辑
- patch的时候，发现这个VNode是组件节点，创建组件App实例，进入组件的渲染过程
- 由于vue-loader会将template中的html转化成render函数，所以生成的App组件渲染VNode为_vnode，上面创建的占位符VNode为$vnode
- 进行App组件的渲染，期间创建子节点时发现了HelloWorld子组件，创建该子组件实例，同样的把该子组件占位符VNode设为$vnode，得到子组件的渲染VNode为_vnode
- HelloWorld子组件的渲染VNode渲染期间再没有嵌套组件，走普通节点的创建流程，patch结束后，将真实DOM返回到HelloWorld组件实例的$el上，同时也映射到其占位符节点的componentInstance上，然后我们在占位符VNode的elm属性得到真实的DOM，插入App组件实例的DOM节点中
- 同理，App组件渲染完毕，插入body中

## 操作DOM的API封装