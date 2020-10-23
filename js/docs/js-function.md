## 基础语法

1.如果函数内部定义了和外部变量同名的内部变量，外部变量被忽略
```
var b = 'a'

function test () {
  var b = 'b'
  console.log(b)
}

test() // 'b'
```

2.函数的参数在执行时会生成传入变量的局部变量副本，所以在函数内部修改参数的值不会影响到传入的变量
```
let a = 'liu'

function test (a) {
  a = 'haha'
}

test()

console.log(a) // 'liu'
```

3.现代JS对未传入值的参数的默认值处理相比原来更有优势
```
function test (a) {
  console.log(a ?? 'haha')
}

test() // 'haha'
test(0) // 0
```

## 函数对象

1.函数被视为一个可调用的行为对象，说明我们可以把它当成对象来处理，比如说增删属性、按引用传递等
```
function test () {}

test.name // 'test'
```

函数的名字属性`name`会根据上下文命名，而不需要在定义时显式指定
```
let test = function () {}

test.name // 'test'

function sayHi (test = function () {}) {
  console.log(test.name)
}

sayHi() // 'test'
```

2.函数还具有`length`属性，它代表函数入参的个数，注意`rest`参数不参与计数
```
function f1 (a) {}
function f2 (a, b) {}
function f3 (a, b, ...rest) {}

console.log(f1.length) // 1
console.log(f2.length) // 2
console.log(f3.length) // 2
```

3.函数可自定义其他属性，一个常见的做法就是重写闭包
```
function makeCount () {
  function counter () {
    return counter.count++
  }

  counter.count = 0

  return counter
}
```

这种方式相比于闭包，优势是我们可以在函数外部修改这个值
```
let counter = makeCount()

counter() // 0

counter.count = 10

counter() // 10
```

4.一个有意思的行为，我们可以为函数表达式添加函数名
```
let sayHi = function func (a) {
  if (a) {
    console.log(a)
  } else {
    func ('Guest')
  }
}

let welcome = sayHi
sayHi = null

welcome() // 'Guest', still work
```

这样做的好处是创建的这个函数名只存在于函数内部，外部不可访问，它确保我们可以递归调用自身，之所以不用`sayHi`就是因为如果`sayHi`这个外部变量被修改时，函数的执行会报错

5.实现一个无限调用的求和函数，调用方式为`sum(5)(-1)(2)(3) === 9`
```
function sum (a) {
  let total = a

  function fn (b) {
    total += b

    return fn
  }

  fn.toString = function () {
    return total
  }

  return fn
}

alert(sum(5)(-1)(2)(3)) // 9
```

6.函数声明和函数表达式的不同

- 函数声明在代码块内部任意位置可见，这意味着我们可以先使用再声明
- 函数表达式只有在执行到时才创建，之后才能正常调用

```
// 函数声明的提前调用
test('a') // 'a'

function test (a) {
  console.log(a)
}

// 函数表达式执行时才创建
test('a') // error, test is not defined

let test = function (a) {
  console.log(a)
}

// 函数声明在代码块外不可见
let age = 16

if (age > 18) {
  test() // 'old'

  function test () {
    console.log('old')
  }
} else {
  function test2 () {
    console.log('young')
  }
}

test() // error, test is not a function
```

7.函数递归，递归就是函数自调用，在函数调用时，引擎会生成一个执行上下文，每个函数有且仅有一个执行上下文，用来保存函数调用时的变量、位置、状态等信息，如果在调用过程中发生递归，就会把当前执行上下文中的变量和执行位置保存到一个`执行上下文堆栈`中，创建一个新的执行上下文放到栈顶，等到当前调用完成后，执行上下文出栈，就会恢复上一次调用的执行上下文，我们就可以重新拿到执行位置和变量

8.尾递归优化，部分引擎实现了尾递归优化，就是说，如果递归发生在函数调用的末尾，在该调用后面不再需要执行其他调用，`执行上下文堆栈`也就不需要保存这个执行上下文，减少内存的消耗

9.能够用递归实现的函数一定能用循环实现，有时候我们会选择循环来优化递归，这是因为递归的特性使得它消耗更多的内存资源，但也有例外，对于某些问题使用循环解决会造成逻辑复杂，分支繁多不便理解，而使用递归在递归深度小于`10000`以内时可以忽略其性能问题，所以这种情况选择递归可以让代码更容易理解，更容易理解的代码意味着更容易维护