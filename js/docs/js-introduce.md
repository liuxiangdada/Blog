## JS的诞生

JS全称`JavaScript`，由任职于网景公司的`Brendan Eich`设计，最初设计的目的是为了解决网页和用户的交互

其实当时的网页只能浏览，如果要让用户输入信息，网页甚至无法判断用户是否真的填写了，只能把网页的内容发回服务器，交由服务器判断，这种做法十分笨重且体验极差，于是就想到使用一种网页脚本语言来处理类似于表单输入提交的简单交互，当时的网景公司有两个选择，一个是直接使用现成的语言，这样做的好处是能够充分利用现成代码和程序员资源；另一个是重新设计一种完全适用的脚本语言，这种做法不需要考虑兼容的问题并有利于后续扩展

最终决定由`Brendan Eich`操刀设计，`Brendan Eich`被要求设计JS使其看起来和`Java`尽可能的像，同时又比Java简单，让网页开发者容易上手，`Brendan Eich`的策略是借鉴

- C语言和Java的基本语法
- Java的数据类型和内存管理
- Scheme的以函数为核心的理念，引入闭包
- Self的基于prototype的继承机制
- Perl的正则表达式
- Python的字符串和数组的处理

站在巨人的肩膀上，使得JS的特性易于使用且灵活，并由此开始了在网页开发中的制霸地位

## JS的发展

JS发展的迅速引起了微软的关注，微软模仿JS开发了一门相近的语言`JScript`，网景公司面临着将要失去网页脚本语言主导权的局面，于是联合国际标准化组织(ECMA)发布了JS的国际标准，并命名为`ECMAScript`

- 1997年，ECMAScript1.0版本发布，文件名为ECMA-262
- 1998年，ECMAScript2.0版本发布
- 1999年，ECMAScript3.0版本发布

在后续的时间里，一直使用ECMAScript3.0，直到2007年开启了对ECMAScript3.0的升级，这期间提出了ECMAScript4.0的草案，由于草案包含的更新过为激进，引发了各方的分歧，终止了4.0的开发，最后的结果是改为发布ECMAScript3.1版本，包含了4.0中一部分对现有功能的改善，其中的大部分功能被推迟到后面的版本计划

ECMAScript3.1这个命名并没有持续很久，在会后被更名为ECMAScript5，在2011年，ECMAScript5.1发布并成为ISO国际标准，直到2012年，主流浏览器都支持ECMAScript5.1的全部功能

2013-2015年期间，举办会议讨论ECMAScript6，该会启用了大部分的新特性并与2015年正式发布，这时ECMAScript的命名规则也改成了以年号结尾，所以ECMAScript6也被成为ECMAScript2015

从ECMAScript6到现在为止，每年ECMA都会根据草案、提议、标准的流程更新JS使之符合开发的需求

## JS的设计缺陷


由于`Brendan Eich`设计的时间很短，导致JS语言存在许多的不合理之处，起初由于其解决了网页的痛点而被广泛关注并使用，规模爆炸式增加使得其迅速被固化下来，由于其历史原因，我们的JS拥有着不少的缺陷，下面我们总结一下

1.null和undefined，这两个类似但又不同的值，null表示空对象，undefined表示变量未定义

```
typeof null // object

typeof undefined // undefined
```

2.==会对比较的双方进行隐式的类型转化且规则不一致

```
'' == false // true
null == undefined // true
0 == '0' // true
0 == '' // true
false == undefined // false
```

3.NaN表示非数值，但它又不等于自身

```
NaN === NaN // false

isNaN(NaN) // true
```

4.万物皆对象，包括数组和函数，使得我们在理解上很难区分

```
typeof [] === 'object'
typeof function () {} === 'function'
```

5.全局变量随处可用，难以控制

```
(function () {
  a = 'global'
})()

console.log(a) // global
```

6.JS不强制要求句尾分号但会自动添加句尾分号

```
function Foo () {
  var tmp = '1'
  return
  {
    name: tmp
  }
}
```

上面的写法不会报错但实际上不会返回对象而是undefined，就是因为自动在return后面加了分号


到目前为止，JS的缺陷仍然存在并将一直存在，随着ECMAScript标准的迭代更新，这些缺陷中的一部分得以被新的特性取代，而另一些缺陷却会一直伴随着JS的发展