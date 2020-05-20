### Node环境的事件循环与浏览器环境的异同

浏览器环境严格遵循宏任务 > 微任务 的流程，而NodeJS环境则是在事件循环的不同阶段执行微任务的，下面详述

```
setTimeout(function() {
  console.log('timer1');
  Promise.resolve().then(function() {
        console.log('promise1')
    })
}, 0)

setTimeout(function() {
  console.log('timer2');
  Promise.resolve().then(function() {
        console.log('promise2')
    })
}, 0)

// 浏览器
timer1
promise1
timer2
promise2

// Node
timer1
timer2
promise1
promise2
```

### NodeJS的EventLoop机制

由于NodeJS运行在服务器系统内核上，所以可以借助系统内核来执行非阻塞I/O操作，当后台执行的多个操作完成后，会通知Node，Node将相应的回调放入任务队列中

Node启动时即开启事件循环，执行顺序如下图：

```
   ┌───────────────────────┐
┌─>│        timers         │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     I/O callbacks     │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
│  │     idle, prepare     │
│  └──────────┬────────────┘      ┌───────────────┐
│  ┌──────────┴────────────┐      │   incoming:   │
│  │         poll          │<─────┤  connections, │
│  └──────────┬────────────┘      │   data, etc.  │
│  ┌──────────┴────────────┐      └───────────────┘
│  │        check          │
│  └──────────┬────────────┘
│  ┌──────────┴────────────┐
└──┤    close callbacks    │
   └───────────────────────┘
```

- timers阶段，执行定时器的回调
- I/O callbacks阶段，执行一些系统调用错误回调，比如网络通信的错误回调
- idle，prepare阶段，仅在node内部调用
- poll阶段，获取完成的I/O事件回调，适当情况下可阻塞Node
- check阶段，执行setImmediate回调
- close callbacks阶段，执行 socket 的 close 事件回调

### 基本流程

- 首先一样在主线程中跑主代码，主代码运行完成
- 进入timer阶段，首先Node会检查设置的定时器有无超时，若超时推入任务队列，有可能设置的时间还未到，所以timer阶段的任务队列为空，进入下一阶段
- I/O callbacks阶段跳过
- 进入poll阶段后，会检查系统的I/O操作哪些完成并将完成的回调推入任务队列，当执行完成后任务队列为空或者回调执行达到上限后进行下一步
- 检查有无预设setImmediate，有的话进入check阶段执行setImmediate回调，没有会等待一段时间，期间等待事件推入队列并执行
- 为防止阻塞，这个阶段EventLoop会检查timer队列是否为空，非空的话会执行timer队列中的回调并返回timer阶段

### nextTick

上面的流程图和示意图都没有提到nextTick，这是因为nextTick独立于事件循环，它不属于事件循环的任何一个阶段，在事件循环的每个阶段，任务队列中的事件执行完毕后，都会执行nextTick任务队列中的事件，并且优先级高于微任务队列

nextTick的这一机制会阻塞事件循环（递归调用自身引发I/O starving），所以应避免使用nextTick，而使用setImmediate

### 总结

- NodeJS的事件循环分为6个阶段
- 浏览器的微任务在每次执行宏任务后执行，NodeJS的微任务在每个阶段执行完毕后执行，举个例子，进入timer队列后，会先执行超时的定时器回调，如果回调中有微任务，推入微任务队列，等到定时器回调执行完毕后顺序执行微任务队列
