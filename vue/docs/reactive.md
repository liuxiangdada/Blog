# 响应式原理

## 响应式对象

核心是`Object.defineProperty`方法，通过挟持对象属性（给对象属性添加get、set方法），每次取值或者改值得时候都执行额外的操作

Vue在初始化过程中，会把props和data都变成响应式对象，如果子属性也是对象则递归把该对象也变成响应式

## 依赖收集

我们知道响应式对象就是挟持对象属性，接下来我们分析get方法中干了什么，在组件挂载过程中会执行render函数将VNode渲染到真实DOM中，这期间就会从data、props中取值，也就是触发了我们设置的get函数，get函数主要做两件事，一件事是返回该属性所对应的值，这也是其本职工作，另一件事就是记录谁调用了该属性，就是谁使用了我，我以后变了该通知谁，这也就是所谓的依赖收集

get的依赖收集实现的很巧妙，借用了Dep这个中间对象，每个被监听的属性都new了这个类并作为闭包保存在内存中，这个dep对象就负责记录哪些watcher使用了该属性，具体做法是调用闭包dep对象的depend方法，在之前Dep类维护了一个全局静态变量`target`，记录当前渲染/计算的watcher，depend方法会把自身作为参数传入Dep.target的addDep方法，然后在当前渲染watcher的addDep方法中将自身添加到闭包dep的subs属性中

听起来有点绕，为啥不直接把Dep.target加到subs中呢，这是因为我们还需要在watcher实例中处理这个dep，举一个很常见的例子，一个数据在模板中多次使用，这样每次使用都会触发addDep，这时就需要在watcher实例中用newDepIds记录当前已经被订阅过的数据，下次跳过避免重复订阅；另一个例子是如果我们进行条件渲染，模板两个地方使用的数据部分不同，就会存在前一次渲染中用到的数据后一次已经用不到的情况，即这些数据的变化已经对渲染无用了，这时就需要移除这些数据上的订阅：具体做法是在cleanupDeps方法中，每次watcher实例收集完依赖时，就判断上次渲染的数据deps中有哪些是这次渲染的数据newDepIds中不存在的，不存在则移除该订阅，并保存当前渲染数据到deps和depIds，最后清空newDeps和newDepIds，方便下次渲染

## 派发更新

setter函数的一个细节，针对没有设置setter方法的属性，Vue直接把新值赋值给`val = obj[key]`的val，这样也实现了对该属性的修改，实际上我们新建一个变量并赋予对象的某个属性值，其实只是赋值的引用，并不是原始值的拷贝，所以修改该变量会同步引发对象属性的修改

派发更新简单来说就是数据修改触发setter，然后调用dep实例的notify通知订阅watcher实例列表更新，每个wacther都有一个update方法，该方法在正常情况下会将自身推入队列queue中，然后在下一个tick触发更新，具体点是执行watcher实例的run方法，其中会调用get方法，该方法会调用传入的updateComponent方法触发重新渲染，关键代码是下面这一句：
```
vm._update(vm._render(), hydrating)
```

注意，其中的_render函数会在生成VNode节点时引用数据，再一次进行依赖收集，其实首次渲染时也会执行进行初次的依赖收集

派发更新使用队列进行了渲染的优化，首先使用has数组记录wathcer的id确保不会重复添加（其实并不能完全避免，后面会提），然后使用waiting变量避免不会重复触发flushSchedulerQueue，这个设计很巧妙，因为JS是单线程的，所以在执行`nextTick(flushSchedulerQueue)`前，把waiting置为true，确保后面调用queueWatcher方法不会再执行nextTick，这样就能把数据的更新渲染统一推迟到下一个tick执行，提高效率

执行flushSchedulerQueue时，首先对queue进行从小到大排序，这是出于先父后子、先用户watcher后渲染watcher、父watcher在run时会销毁子watcher三点考虑的，接着遍历queue，这里做了一个无限渲染判断，即我们思考这样一种情形，在自定义watcher监听某个数据时，其回调再次修改了该数据，这会导致在执行回调时又一次触发queueWatcher并将该用户watcher插入wathcer从而陷入无限循环，Vue在检查到某个wathcer的id被run了100次以上时会主动跳出遍历并报错

## nextTick

## computed和watch