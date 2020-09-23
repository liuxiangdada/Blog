## 引子

this在JS中一直是一个老大难问题，每次我们碰到它时都不免谨小慎微，每次我们自认为理解透彻后，不久又会在生产实践中产生新的困惑，下面我根据自己的理解一步一步带你理解这个神秘的this

## 内存中的函数和对象

不同于原始值，函数和对象在内存中是保存在堆中的，把对象或者函数赋值给变量实际上是把数据结构存在堆，然后将地址作为变量值保存在栈中
```
let a = {}, b = {}

a !== b // true

let c = a

a === c // true

```

复杂点的情况，如果把函数赋值给对象属性，则是把函数的内存地址赋值给该对象属性的属性描述对象的value中
```
var name = 'da'

function f () {
  console.log(this.name)
}

obj = {
  name: 'liu',
  fn: f
}

f() // 'da'

obj.fn() // 'liu'
```

## this的本质

在传统函数中，this指代函数调用时的当前执行上下文，我们之所以困惑，其实是没有搞清楚函数执行时的活动上下文

```
var name = 'liu'

function foo () {
  setTimeout(() => {
    console.log(this.name)
  }, 100)
}

let obj = {
  name: 'xiang',
  fn: foo
}

obj.fn() // 'liu'
```

`obj.fn`的执行上下文显然是obj这个对象，我们执行fn时发现其实际上是一个定时器，定时器在JS中会被推入宏任务队列，这意味着执行fn其实只是把匿名函数推入宏任务队列中，等到下次事件循环执行到匿名函数时，obj上下文已经被弹出执行栈，当前上下文只剩全局对象（在浏览器中是windows）

```
function foo () {
  console.log(this.name)
}

var name = 'liuxiang'

let a = {
  name: 'xiang',
  b: {
    name: 'liu',
    fn: foo
  }
}

a.b.fn() // 'liu'
```

嵌套对象a中的b属性仍为对象，所以我们在`a.b`时实际上是取b属性的描述对象的value中`{ name: 'liu', fn: foo }`这个数据结构的地址，然后根据地址找到b代指的对象，这时的执行上下文已经由a切换至b，所以fn调用时取得b的name属性


## 常见的几种this场景

### 全局调用

```
var name = 'liuxiang'
function foo () {
  console.log(this.name)
}

foo() // 'liuiang'
```

对于纯函数的直接调用，this指向全局对象，在浏览器中为`windows`，严格环境`use strict`下，this指向`undefined`

### 对象方法调用

```
let name = 'haha'
let obj = {
  name: 'liuxiang',
  fn: function () {
    console.log(this.name)
  }
}

obj.fn() // 'liuxiang'
```

函数作为对象的方法调用，其执行环境就是对象本身

### 构造函数调用

```
function Foo () {
  this.name = 'liu'
}

let obj = new Foo()
obj.name // 'liu'
```

使用`new`操作符时，会先在函数内部创建一个空对象，然后把this指向这个对象，接着执行函数内部代码，最后返回这个对象，因此我们可以知道，作为构造函数的this指代new创建的这个对象实例

### 事件绑定

```
<button onclick="onClick()"></button>

function onClick () {
  console.log(this) // windows
}
```

上面的行内绑定形式在按下按钮时会在当前节点寻找onClick函数，找不到会在全局对象中寻找，所以回调是在全局对象中调用的


```
<button id="btn"></button>

let btn = document.querySelector('#btn')

btn.onclick = function () {
  console.log(this) // btn节点对象
}

<!-- 或者写成这样 -->
btn.addEventListener('click', function () {
  console.log(this) // btn节点对象
})
```

对于绑定到节点以on开头的事件属性中的函数，按下按钮时回调作为btn节点对象的方法被调用，指向节点本身

## 箭头函数中的this

箭头函数和普通函数的区别就是箭头函数没有this对象，在箭头函数内部使用this等同于使用一个普通变量，这时，发现当前函数作用域不存在就会向上寻找

箭头函数因为其this的机制类似于变量，所以this的绑定是在定义时固化的，不是在运行时确定

```
var name = 'liu'

function foo () {
  var name = 'haha'
  setTimeout(() => {
    console.log(this.name)
  }, 100)
}

let obj = {
  name: 'xiang',
  fn: foo
}

obj.fn() // 'xiang'
```

上面的例子中，foo函数定时器中使用箭头函数作为回调，在定义时向上寻找this，因为foo函数单独构成作用域，且作为obj的方法被定义，所以foo函数的this指向obj，箭头函数的this和foo保持一致，也指向obj

```
const name = 'liuxiang'
const foo = () => a => void this.name

let obj = {
  name: 'xiang',
  fn: foo
}

obj.fn()() // 'undefined'
```

这里和上面有所不同，原因在于this总是向上寻找某个作用域内的this指向，虽然foo作为obj的方法，但obj单独不构成作用域，所以指向全局，又因为const定义的变量不会挂载到全局，所以输出undefined

## 操纵this的方法

### apply和call

这两个行为类似，都是显式改变this的指向，具体差别见例子
```
function foo (a) {
  console.log(this.name, a)
}

let obj = {
  name: 'liuxiang'
}

let a = foo.apply(obj, [1, 2, 3])
let b = foo.call(obj, 1, 2, 3)
```

### bind

bind单独拿出来说是因为它和上面的两个方法所有不同，它接收一个对象作为函数内部的this并返回一个新函数，其实现本质是内部构造一个新的函数并绑定原函数的this，所以即使我们对这个新函数调用apply和call，也无法改变原函数的this指向，这被称为硬绑定

```
function foo () {
  console.log(this.name)
}

function bindFn (fn, context) {
  return fn.bind(context)
}

var name = 'liu'

let a = {
  name: 'xiang'
}

let b = bindFn(foo, a)

b.apply(null) // 'xiang'
```