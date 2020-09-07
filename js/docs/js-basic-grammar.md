# JS基础知识笔记

## 类型、值和变量

1. JavaScript的类型可分为可变类型和不可变类型，不可变类型如数字、布尔值、null和undefined；可变类型如数组、对象，他们可以修改属性值或者数组元素的值。要注意的是字符串虽然可以看成是字符组成的数组，但它是不可变的，JavaScript并未提供修改已知字符串的方法
2. JavaScript里数字的类型只有Number类型，也就是64位的浮点数，其对整数的表示等同于双精度浮点数对整数的表示
     - 双精度浮点数由1位符号、11位指数、52位尾数构成，计算机储存数字是按照科学计数法来存的，我们知道所有的十位数都可以表示为`1.xxx * 10 ^ n`，但是计算机并不认识十进制，我们只能把数字转化成二进制的科学计数法`1.xxx * 2 ^ n`
     - 十进制浮点数转为二进制时整数部分采用`除2取余，逆序排列`，小数部分采用`乘2取整，顺序排列`，举个例子，8.25转成二进制为1000.01，用科学计数法表示为`1.0001 * 2 ^ 3`，由于指数位是采用移位储存的，指数位都在原数值的基础上加`2 ^ n - 1`进行储存，单精度下指数位为8位，所以指数位的储存值为130，尾数位由于首位都为1，可省略不存，即单精度下储存为`0 10000010 0001...0`
     - 现在我们根据这个来计算最大整数，我们推测出当二进制科学计数法为`1.111...111 * 2 ^ 52`时达到最大，因为再加1则尾数总位数变为53位，超出52位尾数的表示范围了，多出的部分被截断，意思就是我们无法区分`2 ^ 53`和`2 ^ 53 + 1`，也就是不安全了，所以安全整数的范围为`-2 ^ 53 - 2 ^ 53`（不包含边界）
3. 浮点数可以使用指数计数法，具体语法是将10进制科学计数法的10用`E|e`表示，后面跟指数值，如`1.25 * 10 ^ 6`可以表示为`1.25e6` 
4. `Math.log(10)`表示为以自然对数为底10的对数；`Math.log(100)/Math.LN10`表示以10为底100的对数，这是因为Math.LN10为10的自然对数，根据换底公式可得
5. Javascript中的非数字值由一些异常计算得到，比如`0 / 0`，`-1 / Infinity`，`parseInt('abc')`，NaN与任何数都不相等，包括自身，所以判断一个数是否为NaN，可以用isNaN方法
6. Javascript采用IEEE-754浮点数表示法，注意这种表示法无法精确的表示0.1这样简单的浮点数，所以我们会发现`0.1 + 0.2 !== 0.3`这样的现象
7. 关于JS的Date对象，有几点要注意
     - getYear方法获取的是距今的年份差
     - getMonth方法获取的月份数是从0开始数起的
     - getDay方法获取的是星期，并且0表示星期日
8. null是Javascript中的一个关键字，表示一个特殊值，使用typeof求其类型得到object，这表示我们可以将其看成一个特殊对象，意为非对象，通常用来表示数字、字符串和对象是无值的
9. undefined是JavaScript中定义的第二个表示值的空缺的关键字，含义是未定义，可以用来表示一个被定义但没有被初始化、没有返回值的函数返回undefined、引用未提供函数实参的形参的值得到undefined
10. null和undefined的区别：undefined表示系统级的、出乎意料的或者类似错误的值的空缺，而null表示程序级的、正常的或在意料之中的值的空缺
11. 我们知道，像数字、字符串、布尔值这种基本数据类型是原始值的一种，但为什么我们可以使用方法呢，比如说`'abc'.charAt(0)`，原因是JavaScript发现引用了字符串的属性时吗，会自动调用其包装对象String，将其转化成对象，并调用这个临时对象的方法，结束后销毁，这也是为什么我们调用undefined或者null的属性或方法时报错的原因，因为它们没有包装对象