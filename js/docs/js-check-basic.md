## 数据

### 字符串

1.我们都知道在`ES6`中新增了反引号用来表示多行字符串，除了在内部使用`${}`将任何表达式嵌入字符串中的用法，反引号实际上还支持模板函数，通过在第一个反引号前指定函数名，该函数就会被自动调用并传入模板字符串作为参数
```
function print(str, val) {
  console.log(str.join(' ') + val)
}

print`Hello${1}World` // 'Hello World1'
```

2.常见的操作字符串的方式
```
let str = 'abcd'

str.length // 4

str[0] // 'a'

str.charAt(1) // 'b'

for (let s of str) console.log(s) // 'a', 'b', 'c', 'd'

```

3.字符串虽然和数组类似，比如可通过`length`获取长度，可遍历，但我们要注意字符串本身是不可变的，单纯的改变字符串中某个位置的值没有效果，如果我们确实需要修改字符串，好的做法是把处理的结果存放到一个新建的字符串变量中
```
let str = 'Hello'

str[0] = 'h'

console.log(str) // 'Hello'

let str2 = 'h' + str.slice(1) // 'hello'
```

4.对于字符串查找，我们通常使用`indexOf`方法，找不到的情况该方法会返回`-1`，无法直接在判断语句中使用，这里介绍一种技巧`按位NOT`，`~`运算符会将数字转为32位整数并对其二进制位中的所有位取反，通俗一点就是`~n === -(n + 1)`，利用这点我们可以优化字符串的查找判断
```
let str = 'abcd'

if (~str.indexOf('a')) {
  // find out
}
```

5.所有的字符串都使用`UTF-16`编码，每个字符都有对应的数字编码，通过特殊的方法我们可以实现数字代码到字符的相互转化
```
let str = '1234'

console.log(str.codePointAt(0)) // 49

String.fromCodePoint(49) // '1'
```

实际上我们原来也有`str.charCodeAt`和`String.fromCharPoint`两个方法，定义新的方法是为了处理代理对（所谓代理对就是一些稀有字符无法用2字节表示，而是使用一对2字节的符号编码），老的方法由于代理对在JS创建时并不存在，所以无法正确处理
```
let str = '𝒳'

console.log(str.length) // 2

str.charCodeAt(0) // 55349

String.fromCharCode(55349) // '�'

// 正确的处理

str.codePointAt(0) // 119987

str.codePointAt(1) // 56499，和老的方法一样取代理对的后一个字节的数字编码

String.fromCodePoint(119987) // '𝒳'
```

6.字符串中有一些字符含有音标，和原字符不同，这些变音符号允许使用多个`unicode`字符表示（当然一些常见的变音字符也可以拥有单字符的数字编码），这带来一些表示和比较上的问题，所幸后来我们定义了一些方法对其进行处理
```
let str1 = 'S\u0307\u0323' // Ṩ
let str2 = 'S\u0323\u0307' // Ṩ

str1 === str2 // false

str1.normalize() === str2.normalize() // true， '\u1e68'

'Ö' > 'Z' // true
'Ö'.localeCompare('Z') // -1，代表'Ö'小于'Z'，这符合直觉
```

### 变量

1.`var`、`let`、`const`的区别

- `var`没有块级作用域，不是函数作用域就是全局作用域
- `var`允许重复声明
- `var`声明的变量可以在其声明语句前被使用

```
// 神奇的情况

if (0) {
  var a = 1 // 这里的逻辑理论上永远不会执行，但a还是被声明了
}

console.log(a) // undefined
```

在旧时代，我们对块级作用域的`hack`是`IIFE`立即执行函数，随着`let`和`const`的面世，它的作用已经越来越小了

### 数据类型

1.`null`和`undefined`的各种表现

在`==`中`null`和`undefined`不能转为其他类型，所以我们看到下面的结果
```
null == undefined // true

null == 0 // false

null == '' // false

undefined == 0 // false

undefined == '' // false
```

在运算符中，`null`被视为`0`，而`undefined`被视为`NaN`
```
1 - null // 1

null >=0 // true

undefined + 1 // NaN

undefined >= 0 // false

```

2.`NaN`的各种表现

`NaN`不等于自身，只能通过`isNaN`判断一个值是否是非数字
```
NaN == NaN // false

isNaN('a') // true
```

`NaN`与其他数据做加减乘除均得到`NaN`，`NaN`与其他数据比较均得到`false`
```
1 + NaN // NaN

1 * NaN // NaN

NaN >= 0 // false

NaN < -1 // false
```

### 数字类型

1.我们可以把一些长数字通过简写来避免写一长串的0，不仅是因为懒，更多的是出于避免错误的考虑
```
10000 === 1e4
10000 === 10e3

10007 === 1e4+7

0.001 == 1e-3 // 3表示除1外0的数量
```

2.一般数字在JS中有多种进制的表示，比如说当需要按位操作时我们用二进制表示；颜色代码我们用16进制表示，`toString()`方法接收参数`base`，指定转化为哪种进制的表示字符串
```
let num = 255

num.toString(2) // '11111111'

num.toString(8) // '377'

num.toString(16) // 'ff'
```

当然我们所熟知的还有`parseInt`方法，该方法接收第二个参数表示用什么基数来解析传入的字符串
```
parseInt('10') // 10

parseInt('11111111', 2) // 255

parseInt('377', 8) // 255

parseInt('ff', 16) // 255

parseInt('0xff', 16) // 255，parseInt方法内部对16进制做了特殊处理使得其能够识别0x前缀
```

3.数字的舍入方式总结

`Math`数学方法库的四个常用方法`round`、`floor`、`ceil`、`trunc`
```
Math.round(1.5) // 2，四舍五入
Math.floor(1.5) // 1，向下舍入
Math.ceil(1.5) // 2，向上舍入
Math.trunc(1.5) // 1，移除小数点后的内容
```

一些操作会把小数先转化成整数再处理，比如说位运算符，这些位运算符会直接丢弃掉小数后面的部分
```
4.9 >> 1 // 2
3.9 >> 1 // 1
```

4.保留小数

用的最多的是`toFixed`方法，它把数字保留指定位数（四舍五入）的小数并转成字符串返回
```
'1.2252'.toFixed(2) // '1.23'
```

我们也可以用乘除法模拟
```
let num = 1.23456

console.log(Math.floor(num * 100) / 100) // 1.23
```

此外还有`toPrecision`和`toExponential`方法
```
let num = 1.23456

console.log(num.toPrecision(4)) // 1.235
console.log(num.toExponential(3)) // 1.235e+0
```

5.小数的不精确性，就好像我们在十进制中无法表示`1/3`这种无限循环小数一样，在计算机中，我们也无法表示诸如`0.1`、`0.2`这样的小数
```
0.1 + 0.2 === 0.3 // false

console.log(0.1 + 0.2) // 0.30000000000000004
```

通过`toFixed`方法我们可以看到精度的损失，在运算中这个损失会累加
```
0.1.toFixed(20) // '0.10000000000000000555'

0.2.toFixed(20) // '0.20000000000000001110'
```

所以我们要避免对小数进行比较，或者把小数的运算转化为整数的运算，避免精度的损失

6.处理数字判断的历史问题`Object.is`，我们原来是无法区分`NaN`和正负0的，使用该方法可以得到正确的结果
```
NaN === NaN // false

0 === -0 // true

Object.is(NaN, NaN) // true
Object.is(0, -0) // false
```

7.奇葩问题，为啥`6.35.toFixed(1) == 6.3`，这个就是一个典型的精度损失的问题，我们把`6.35`打印出来就一目了然了
```
6.35.toFixed(1) == 6.3 // true

console.log(6.35.toFixed(20)) // 6.34999999999999964473
```

8.生成从`min`到`max`的随机整数，由于四舍五入的存在，我们得到的整数的概率是不一样的，比如下面这个例子
```
function random (min, max) {
  return Math.round(min + (max - min) * Math.random())
}

random(1, 3)

得到1的值的范围是`1-1.4999999...`
得到2的值的范围是`1.5-2.4999999...`
得到3的值的范围是`2.5-3`

所以得到2的概率是其他两个值的两倍
```

我们可以采用取`min - 0.5 ~ max + 0.5`的方法
```
function random (min, max) {
  return Math.round((min - 0.5) + (max - min + 1) * Math.random())
}
```

也可以采用`Math.floor`的方式取`min ~ max + 1`的范围
```
function random (min, max) {
  return Math.floor(min + (max - min + 1) * Math.random())
}
```

### 类型检测

除了`typeof`和`instanceof`之外，我们还介绍过`Object.prototype.toString`方法来检查数据类型，但这里我们扩展一下，`toString`方法实际上会查找目标对象的`Symbol.toStringTag`属性，拿到其值作为`toString`的返回值
```
let obj = {
  [Symbol.toStringTag]: 'User'
}

obj.toString() // '[object User]'
```

## 运算

### 值的比较

1.字符串之间的比较是按顺序比较每一位的数字编码，直到比较出结果或者一方没有可比较的字符
```
'abcd' > 'abcc' // true

'abcd' > 'b' // false

'abcd' > 'abc' // true

'abcd' === 'abcd' // true
```

### 运算符

1.运算符的优先级

- 乘除优先级高于加减
- 括号优先级最高
- 一元运算符高于二元运算符

2.我们知道`=`有赋值功能，但我们经常会忽略它的另一个作用，返回赋值的结果
```
let a = 0

let c = 3 - (a = 2 + 1) // 0
```

`=`运算符还支持链式操作，使得多个变量共享同一个值
```
let a, b, c

a = b = c = 0
```

3.自增/自减操作有前置和后置两种形式，它们之间的区别是返回值的不同
```
let a = 2

a++ // 2，先返回再自增
++a // 4，先自增再返回
```