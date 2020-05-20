### 什么是异步？和同步有什么区别？

了解异步之前我们先了解一下同步，在日常生活中，同步是多个事情或者人按照相同的节奏完成，看起来步调一致，而在 JS 中，同步有些许不同，它代表程序在执行时处于同一流程，也就是说前面执行的结果可以立刻被后面的代码取得并使用

由于 JS 是浏览器的脚本，处理网络请求数据是必须具备的能力，而网络请求受多方限制，不可能立即得到结果，如果浏览器在渲染页面时被某个请求阻塞，就会造成页面白屏或者加载不完全，用户体验很差

很自然，我们会想到对于任务进行分类，对于那些可以直接得到结果的任务，先执行，而对于需要等待结果的任务，在得到结果后再执行，这就是同步任务和异步任务

### 为什么要设计异步？

众所周知，JS 是单线程的，这意味着我们执行 JS 代码时只能按部就班的从上往下顺序执行，如果某一项任务十分耗时，程序就会被堵塞，既然单线程会存在堵塞的风险，JS 为什么不选择多线程运行呢？

多线程，意味着可以同时进行多个任务，而对于 JS 这种能够修改 DOM 的脚本来说，如果同一时刻多个线程中的任务修改了同一处的 DOM 文本，那最后以谁为准呢，显然多线程的弊端更加明显

既然要使用单线程又不能阻塞主线程执行，于是 JS 引入了任务队列的概念，启动 JS 的运行环境后，会创建 3 个区域，分别是堆、栈和任务队列，堆用来存放大型对象等变量，栈主要执行函数并存放临时变量，而任务队列就是存放着一些异步任务的回调事件

有了任务队列后，执行流程就变得清晰了，我们首先顺序执行代码，碰到异步任务，在任务队列中注册事件，将异步的回调放入这个事件中，然后继续往下执行，直到执行栈为空，就去任务队列中查看是否有待执行的事件，有的话取出来执行，直到任务队列为空

### 异步和 EventLoop 的关系？

查看我其他博文对 EventLoop 的详细分析

#### 浏览器

EventLoop 在浏览器中将任务分为宏任务和微任务，宏任务诸如加载的一段 JS 主代码，定时器，页面渲染等；微任务为 promise 及其衍生技术、ajax 请求

执行的顺序为：宏任务 > 微任务 > 重新渲染 > 下一个宏任务

看下面这个例子，试着想一下输出是什么

```
console.log(1)
setTimeout(() => {
  console.log(2)
}, 0)
console.log(3)
```

结果是`1、3、2`，为什么不是`1、2、3`呢？我们试着分析下：这段代码定义了一个定时器，要求延时执行其回调，JS 引擎从上到下执行到定时器时，发现需要被延时 0ms，将其回调扔进任务队列，往下执行，等到执行完毕后取出任务队列中的代码执行

我们再来看一下下面的例子：

```
console.log(1)

Promise.resolve(1).then(() => {
  console.log(2)
})

setTimeout(() => {
  console.log(3);
  Promise.resolve(1).then(() => {
    console.log(4)
  })
}, 0)

console.log(5)
```

相比于前面的例子，更加复杂了，没事我们继续分析一下，首先顺序执行，输出`1`，接着碰到 promise 将 then 后的回调扔进任务队列的微任务队列，继续往下执行，碰到定时器，将其回调扔入宏任务队列，往下执行输出`5`，主代码执行完毕，执行栈为空，去任务队列检查，首先检查微任务队列，发现刚刚加入的回调，输出`2`，微任务队列被清空，检查宏任务队列，发现定时器回调，输出`3`，又发现 promise，再次扔进微任务队列，定时器宏任务执行完毕，执行栈为空，再次进入任务队列检查，发现只有微任务队列存在待执行任务，执行并输出`4`，所以结果为`1、5、2、3、4`

下面我们再看一个经典的面试题，作为练习，不明白可以回去再看上面的两个例子

```
for(var i = 0; i < 6; i++) {
  setTimeout(() => {
    console.log(i);
  }, i*100)
}
```

我们将循环拆解

```
setTimeout(() => {
  console.log(i);
}, 0*100)

...

setTimeout(() => {
  console.log(i);
}, 5*100)
```

上面定时器中回调中的`i`我没有带入值，这里我们会存在一个误区，认为这时的`i`是运行时的值，其实并不是的，根据上面的流程分析，执行主代码时，定义了 6 个定时器，分别延时从 0 到 500ms，这时，主代码执行完毕，`i`的值已被更新为 6，所以等到去执行任务队列中的任务时，输出全为 6

#### Node

为什么要区分一下浏览器和 Node 呢？这时因为 Node 因为服务端支撑需要，对任务执行的要求不像浏览器环境，并且在服务端可以调动系统的多线程，完成更强大的功能

EventLoop 在 Node 中主要有 6 个阶段：timer 阶段、IO 异常 callback 阶段、idel/prepare 阶段、poll 阶段、check 阶段、close callback 阶段

- timers 阶段，执行定时器的回调
- I/O callbacks 阶段，执行一些系统调用错误回调，比如网络通信的错误回调
- idle，prepare 阶段，仅在 node 内部调用
- poll 阶段，获取完成的 I/O 事件回调，适当情况下可阻塞 Node
- check 阶段，执行 setImmediate 回调
- close callbacks 阶段，执行 socket 的 close 事件回调

Node 端同样存在微任务队列，并且微任务队列会在上面每个阶段执行完成后执行

此外，Node 端还存在着`process.nextTick`API 和`setImmediate`API，前者会在上面每个阶段执行完毕后微任务队列执行前执行；后者则是固定在 check 阶段执行

来个例子：

```
setTimeout(() => {
  console.log(1);
  Promise.resolve(1).then(() => {
    console.log(2)
  })
}, 0)

setTimeout(() => {
  console.log(3);
  Promise.resolve(1).then(() => {
    console.log(4)
  })
}, 0)
```

这段代码在浏览器中毫无疑问输出`1、2、3、4`，但在 Node 中，却会输出`1、3、2、4`，这是因为主代码执行时，发现两个定时器，进入 timer 队列，依次执行两个定时器的回调，timer 队列执行完毕后，执行微任务队列中的两个任务

再来一个例子

```
setImmediate(() => {
  console.log(1)

  setTimeout(() => {
    console.log(2)
  }, 0)

  setImmediate(() => {
    console.log(3)
  })

  process.nextTick(() => {
    console.log(4)
  })
})
```

我们还是老样子根据 Node 端的规则进行分析，首先进入 timer 阶段，检查有无超时定时器，发现没有，进入微任务队列和 nextTick 队列也没有任务，进入下一个阶段，中间略去无关部分，进入 poll 阶段，检查 IO 回调发现也没有，于是检查有无 setImmediate，发现存在进入 check 阶段，执行 setImmediate 回调，输出 1，接着碰到定时器，推入 timer 阶段的任务队列，然后碰到 setImmediate，推入 check 阶段的任务队列，发现 nextTick 事件，推入 nextTick 的任务队列，setImmediate 回调执行完毕，执行 nextTick 队列任务，输出 4，重新回到 timer 阶段，执行其任务队列，输出 2，后面跳过无关部分，又进入 poll 队列检查 setImmediate 后执行其回调，输出 3，所以结果为`1、4、2、3`

后面再测试的时候，发现了`1、4、3、2`的结果，这是为什么呢？原因出在我设置的这个定时器上，我设置的是 0ms 的延迟，但是 nodeJS 引擎会强制改为 1ms[官方文档说明](https://nodejs.org/api/timers.html#timers_settimeout_callback_delay_args)，而 1ms 的延迟就取决于同步代码运行的快慢了，如果运行的慢，延迟已经过去了，timer 阶段的任务队列已经存在任务回调就先输出 2，否则先输出 3

如果放在异步回调中就没有这个问题，因为异步的回调一般都在 poll 阶段执行，而 poll 执行完毕后下一个阶段就是 setImmediate，所以 setImmediate 先执行，下面输出`1、3、2`

```
const fs = require('fs')

fs.readFile('test.txt', () => {
  console.log('1')
  setTimeout(() => {
    console.log('2')
  }, 0)
  setImmediate(() => {
    console.log('3')
  })
})
```

### 监听事件是异步吗？

我们在操作 DOM 时会给按钮或者其他交互元素绑定用户事件，等到用户触发时执行绑定的回调，那么有一个问题，这个绑定的回调是异步事件吗？

根据浏览器的 EventLoop 机制，在执行事件绑定时，回调同样会被推入任务队列，区别是其他异步操作是系统自动调用，而绑定事件时用户主动触发，采取的是发布订阅模式，个人倾向于觉得事件绑定不是异步事件

通过一个简单的测试,输出为`1、2`

```
$('#btn').addEventListener('click', function() {
  console.log(1);
})
$('#btn').click();
console.log(2);
```

查阅资料了解到，虽然 JS 是单线程的，但是浏览器不是单线程工作的，内部有多个线程，渲染引擎、JS 解析引擎、I/O 操作、定时器计时、事件监听等使用多个线程处理

所以上面代码的原理大致是 JS 引擎解析时发现事件绑定的回调，将其推入任务队列，然后继续执行，下面触发事件，被其他线程监听到，直接执行回调输出 1，接着继续执行下面的代码输出 2

### 什么是协程？Generator 函数是如何实现的？

协程是一个比线程更小的概念，我们通常熟知进程和线程，一个进程可运行多个线程，但是一个线程内部虽然能存在多个协程，却只能同时运行一个协程，即如果线程内部需要切换协程时，由某个协程交出执行权，暂停执行状态，让下一个协程执行，接着又由其交还执行权，继续执行

Generator 函数正是借鉴了协程的原理，Generator 通过内部的 yield 交出执行权并将 yield 后的值作为结果返回，外部通过 next 方法调用取得一个对象，包含的 value 属性就是 yield 的返回值，附带的 done 属性表示是否执行完毕，那么如何交还执行权呢，答案是 next 方法，调用方法不仅能取得 yield 的返回值，并且交还执行权给内部，可携带参数，在内部通过`let res = yield 100`获取，要注意是给上一个 yield 赋值

```
function* gen() {
  console.log(1)
  const a = yield 1;
  console.log(a)
  return a + 1
}

let g = gen();
g.next(); // 1
g.next(2); // 3
```

### Generator 函数是如何改变异步的写法的？

首先我们要了解一下 thunk 函数，所谓的 thunk 函数就是生成一系列相似功能定制化函数的函数，简单来说 thunk 函数能够将多参数函数简化成较少参数的函数

```
function isString(str) {
  return Object.prototype.toString().call(str) === "[object String]";
}

function isType(type) {
  return fucntion(obj) {
    return Object.prototype.toString().call(str) === "[object " + type + "]";
  }
}

let isString = isType("String");
```

上面的 isType 就是一个 thunk 函数，了解了这个概念后我们先做一些前置工作

```
const fs = require("fs");

function readFileThunk(fileName) {
  return (callback) => {
    fs.readFile(fileName, callback);
  }
}

function* gen() {
  const data1 = yield readFileThunk("1.txt");
  console.log(data1.toString());
  const data2 = yield readFileThunk("2.txt");
  console.log(data2.toString());
}

```

定义好 thunk 函数和 generator 函数后，我们封装一个自执行函数

```
function run(gen) {
  let g = gen();
  g.next().value((err, data1) => {
    g.next(data1).value((err, data2) => {
      g.next(data2);
    })
  })
}

run(gen);
```

最后打印出如下结果：

```
1.txt的结果
2.text的结果
```

不使用 thunk 函数，也可以使用 Promise：

```
const fs = require("fs");

function readFilePromise(fileName) {
  return new Promise((resolve, reject) => {
    fs.readFile((err, data) => {
      if(data) {
        resolve(data)
      } else {
        reject(err)
      }
    })
  })
}

function* gen() {
  const data1 = yield readFilePromise("1.txt");
  console.log(data1.toString());
  const data2 = yield readFilePromise("2.txt");
  console.log(data2.toString());
}
```

执行代码为：

```
function run(gen) {
  let g = gen();

  function getGenPromise(gen, data) {
    return gen.next(data).value;
  }

  getGenPromise(g).then(data1 => {
    return getGenPromise(g, data1);
  }).then(data2 => {
    return getGenPromise(g, data2);
  })
}

run(gen);
```

提到 Generator 就不得不介绍大名鼎鼎的`co`库，co 配合 Generator 使用起来十分的丝滑：

```
const co = require("co");

let g = gen();
co(g).then(res => {
  console.log(res)
})
```

co 内部正是对 Generator 自动执行器的封装，在上面我们实现的原理上，加上了对边界值的判断，下面附上核心源码：

```
function co(gen) {
  var ctx = this; // 获取上下文

  // 返回promise对象
  return new Promise(function(resolve, reject) {
    // 判断是不是Generator函数，不是直接resolve
    if (typeof gen === 'function') gen = gen.call(ctx);
    if (!gen || typeof gen.next !== 'function') return resolve(gen);

    // 封装Generator遍历器对象的next方法，主要增加错误捕获机制
    onFulfilled();

    function onFulfilled(res) {
      var ret;
      try {
        ret = gen.next(res);
      } catch (e) {
        return reject(e);
      }
      next(ret);
    }

    function onRejected(err) {
      var ret;
      try {
        ret = gen.throw(err);
      } catch (e) {
        return reject(e);
      }
      next(ret);
    }

    // 核心方法，会不断调用自身直到遍历结束
    function next(ret) {
      if (ret.done) return resolve(ret.value); // 判断next方法返回结果的done属性
      var value = toPromise.call(ctx, ret.value); // 将得到的值包装成promise对象
      // 持续调用自身
      if (value && isPromise(value)) return value.then(onFulfilled, onRejected);
      return onRejected(
        new TypeError(
          'You may only yield a function, promise, generator, array, or object, '
          + 'but the following object was passed: "'
          + String(ret.value)
          + '"'
        )
      );
    }
  });
}
```

### Async 函数对 Generator 的改进

ES6 之后推出的 async/await 语法，实现了对异步代码的同步化写法，其本质是 generator 的语法糖，将`function*`替换成`async function`；将`yield`替换成`await`

async 语法的优势是语义更加直白，能够从单词内容明白其作用，相比于 generator 的\*和 yield 难以理解；其次 generator 函数不能自动执行，需要自己手写或者调用库来实现自动化，而 async 内置自动执行器，让用户不必了解内部复杂的流程，开袋即食

### 异步代码写法的演变

最后我们总结一下前端程序猿们对异步代码写法的改进

```
// jquery时代
$.ajax(
  url1,
  param,
  success: function(res) {
    var param = res;
    $.ajax(
      url2,
      param,
      success: function(res) {
        console.log(res)
      })
  },
  fail: function(err) {
    console.log(err)
  })
```

```
// promise时代
function ajaxPromise(url, param) {
  return new Promise((resolve, reject) => {
    $.ajax(url, param, success: res => resolve(res), fail: err => reject(err);)
  })
}

ajaxPromise(url1, param).then(res => {
  return ajaxPromise(url2, res);
}).then(res => {
  console.log(res);
})
```

```
// generator时代
function ajaxPromise(url, param) {
  return new Promise((resolve, reject) => {
    $.ajax(url, param, success: res => resolve(res), fail: err => reject(err);)
  })
}

function* gen() {
  let data1 = yield ajaxPromise(url1, param);
  console.log(data1);
  let data2 = yield ajaxPromise(url2, data1);
  console.log(data2);
}

function run(gen) {
  let g = gen();

  g.next().value.then(res1 => {
    return g.next(res1).value;
  }).then(res2 => {
    return g.next(res2).value;
  })
}

run(gen);
```

```
// async 时代
function ajaxPromise(url, param) {
  return new Promise((resolve, reject) => {
    $.ajax(url, param, success: res => resolve(res), fail: err => reject(err);)
  })
}

async function ajaxAsync() {
  let data1 = await ajaxPromise(url1, param);
  console.log(data1);
  let data2 = await ajaxPromise(url2, data1);
  console.log(data2);
}
```
