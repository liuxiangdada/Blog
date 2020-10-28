## 原理

Promise的本质是一种有限状态机，有三种状态，分别为pending、fulfilled、rejected，规定Promise默认处于pending状态，我们可以将其置为fulfilled或者rejected二者之一，状态只能从pending开始改变并且一旦改变便不可再更改

Promise被设计来处理异步问题，我们通常在定义时传入以两个内置函数resolve和reject为参数的函数，该函数通常为一些异步操作，等到未来某一时刻异步完成后，通过resolve和reject修改Promise的状态并返回异步结果，接着我们可以通过Promise原型上的then、catch、finally方法传入处理函数

Promise之所以能够取代原有的回调函数模式，是因为其使得我们不必将回调函数写在异步操作内部，而是等到调用then方法的时候才传入回调，光这一点还不够，Promise所支持的链式调用和错误冒泡让我们可以将异步结果层层处理并统一处理错误

## 实现

通过Promises/A+规范，我们尝试着手写实现一个promise

```
function promise(fn) {
  // 定义三种状态
  const PENDING = "pending";
  const FULFILLED = "fulfilled";
  const REJECTED = "rejected";

  // 初始化内部变量
  let self = this;
  self.status = PENDING;
  self.value = undefined;
  self.error = undefined;
  self.onFulFilled = undefined;
  self.onRejected = undefined;

  function resolve(val) {
    if(self.status !== PENDING) return;
    setTimeout(() => {
      self.status = FULFILLED;
      self.value = val;
      self.onFulFilled(val);
    })
  }

  function reject(err) {
    if(self.status !== PENDING) return;
    setTimeout(() => {
      self.status = REJECTED;
      self.error = err;
      self.onRejected(err);
    })
  }

  fn(resolve, reject);

  promise.prototype.then = function(onFulFilled, onRejected) {
    let self = this;
    if(self.status === PENDING) {
      self.onFulFilled = onFulFilled;
      self.onRejected= onRejected;
    } else if (self.status === FULFILLED) {
      onFulFilled(self.value);
    } else {
      onRejected(self.error);
    }

    return self;
  }
}
```

上面的简版实现实现了Promise的状态修改和延时绑定回调的功能，还有几个问题：

- 无法处理多个回调函数，前者的回调会被后一个覆盖
- 无法进行链式调用，因为返回的是同一个promise，无法将前者的处理结果返回给后者
- 错误处理机制不完善，不能层层冒泡到最后

我们一步步来修改，首先解决处理多个回调的问题，这个问题好改

```
// 首先将内部的回调方法改为数组
self.onFulFilled = [];
self.onRejected = [];

// 在resolve和reject函数中批量处理回调
self.onFulFilled.forEach(f => f());
self.onRejected.forEach(f => f());

// 在then方法中将回调函数存入数组
self.onFulFilled.push(onFulFilled);
self.onRejected.push(onRejected);

```

接着处理一下链式调用的问题，分析一下，要想连续调用then方法，必须使得then方法返回一个新的promise，并且返回的这个promise的状态受前一个影响，即在不传入then参数的情况下下一个promise要能够继承上一个promise的状态，我们来实现一下

```
promise.prototype.then = function(onFulFilled, onRejected) {
  // 处理不传then参数的情况，解决了透传问题
  onFulFilled = typeof onFulFilled === "function" ? onFulFilled : x => x;
  onRejected = typeof onRejected === "function" ? onRejected : e => { throw e; };

  let promise2,
    self = this;

  if (self.status === PENDING) {
    promise2 = new promise((resolve, reject) => {
      self.onFulFilled.push(() => {
        try {
          let x = onFulFilled(self.value);
          resolve(x);
        } catch (e) {
          reject(e);
        }
      });

      self.onRejected.push(() => {
        try {
          let x = onRejected(self.error);
          resolve(x);
        } catch (e) {
          reject(e);
        }
      });
    });
  } else if (self.status === FULFILLED) {
    promise2 = new promise((resolve, reject) => {
      setTimeout(() => {
        try {
          let x = onFulFilled(self.value);
          resolve(x);
        } catch (err) {
          reject(err);
        }
      });
    });
  } else {
    promise2 = new promise((resolve, reject) => {
      setTimeout(() => {
        try {
          let x = onRejected(self.error);
          resolve(x);
        } catch (err) {
          reject(err);
        }
      });
    });
  }
  return promise2;
};
```

到这里看似很完美了，但是我们忽略了回调函数返回值为promise等复杂值的情况，而是直接将其resolve了，根据Promises/A+规范，我们首先实现一个promise处理函数

```
  // promise解决程序
  function resolvePromise(promise2, x, resolve, reject) {
    // x不能和promise2相同，避免循环引用
    if (promise2 === x) return reject(new TypeError("same promise"));

    // 如果x为promise
    if (x instanceof promise) {
      // 继续拆解
      x.then(
        val => {
          resolvePromise(promise2, val, resolve, reject);
        },
        err => {
          reject(err);
        }
      );
    }
    else if ((typeof x === "object" && x !== null) || typeof x === "function") {
      try {
        let then = x.then;
        if (typeof then === "function") {
          let called = false; // reject或者resolve执行过的话，忽略其他
          try {
            then.call(
              x,
              y => {
                if (!called) {
                  resolvePromise(promise2, y, resolve, reject);
                  called = true;
                }
              },
              r => {
                if (!called) {
                  reject(r);
                  called = true;
                }
              }
            );
          } catch (err) {
            if (!called) {
              reject(err);
            }
          }
        } else {
          resolve(x);
        }
      } catch (err) {
        reject(err);
      }
    } else {
      resolve(x);
    }
  }
```

然后修改一下then方法中的引用

```
let x = onFulFilled(self.value);
resolvePromise(promise2, x, resolve, reject);
```

到这里就算实现了Promise的完整功能了，附上源码：

```
/**
 * 模拟实现promise
 * 根据promise /A+规范
 * 已通过promises-aplus-tests所有测试
 */

function promise(fn) {
  //
  // 定义三种状态
  const PENDING = "pending";
  const FULFILLED = "fulfilled";
  const REJECTED = "rejected";

  let self = this; // 保存当前promise实例
  self.value = null; // 2.1.2.2
  self.error = null; // 2.1.3.2
  self.status = PENDING;
  self.onFulFilled = []; // 2.2.6
  self.onRejected = []; // 2.2.6

  const resolve = function(val) {
    if (self.status !== PENDING) return;

    // 2.1.2
    setTimeout(() => {
      self.status = FULFILLED;
      self.value = val;
      self.onFulFilled.forEach(f => f()); // 2.2.6.1
    });
  };

  const reject = function(err) {
    if (self.status !== PENDING) return;

    // 2.1.3
    setTimeout(() => {
      self.status = REJECTED;
      self.error = err;
      self.onRejected.forEach(f => f()); // 2.2.6.2
    });
  };

  try {
    fn(resolve, reject);
  } catch (err) {
    throw new TypeError("Promise resolver undefined is not a function");
  }

  // promise解决程序
  function resolvePromise(promise2, x, resolve, reject) {
    // 2.3.1，x不能和promise2相同，避免循环引用
    if (promise2 === x) return reject(new TypeError("same promise"));

    // 如果x为promise，2.3.2
    if (x instanceof promise) {
      // 继续拆解，2.3.2.1
      x.then(
        val => {
          resolvePromise(promise2, val, resolve, reject);
        },
        err => {
          reject(err);
        }
      );
    }
    // 2.3.3
    else if ((typeof x === "object" && x !== null) || typeof x === "function") {
      try {
        let then = x.then; // 2.3.3.1
        // 2.3.3.3
        if (typeof then === "function") {
          let called = false; // reject或者resolve执行过的话，忽略其他
          try {
            then.call(
              x,
              y => {
                // 2.3.3.3.3
                if (!called) {
                  resolvePromise(promise2, y, resolve, reject); // 2.3.3.3.1
                  called = true;
                }
              },
              r => {
                // 2.3.3.3.3
                if (!called) {
                  reject(r); // 2.3.3.3.2
                  called = true;
                }
              }
            );
          } catch (err) {
            // 2.3.3.3.4.1
            if (!called) {
              reject(err);
            }
          }
        } else {
          resolve(x); // 2.3.3.4
        }
      } catch (err) {
        reject(err); // 2.3.3.2
      }
    } else {
      // 不是promise直接resolve
      resolve(x); // 2.3.4
    }
  }

  // 2.2
  promise.prototype.then = function(onFulFilled, onRejected) {
    // 处理不传then参数的情况，解决了透传问题
    onFulFilled = typeof onFulFilled === "function" ? onFulFilled : x => x; // 2.2.7.3
    onRejected =
      typeof onRejected === "function"
        ? onRejected
        : e => {
            throw e;
          };

    let promise2, // 2.2.7
      self = this;

    if (self.status === PENDING) {
      promise2 = new promise((resolve, reject) => {
        self.onFulFilled.push(() => {
          try {
            let x = onFulFilled(self.value); // 2.2.7.1
            resolvePromise(promise2, x, resolve, reject);
          } catch (e) {
            reject(e); // 2.2.7.2
          }
        });

        self.onRejected.push(() => {
          try {
            let x = onRejected(self.error);
            resolvePromise(promise2, x, resolve, reject);
          } catch (e) {
            reject(e);
          }
        });
      });
    } else if (self.status === FULFILLED) {
      promise2 = new promise((resolve, reject) => {
        // 2.2.4 异步执行rejected和resolved
        setTimeout(() => {
          try {
            let x = onFulFilled(self.value); // 2.2.7.3
            resolvePromise(promise2, x, resolve, reject);
          } catch (err) {
            reject(err);
          }
        });
      });
    } else {
      promise2 = new promise((resolve, reject) => {
        setTimeout(() => {
          try {
            let x = onRejected(self.error); // 2.2.7.4
            resolvePromise(promise2, x, resolve, reject); // 这里保证了错误被处理后不会继续透传
          } catch (err) {
            reject(err);
          }
        });
      });
    }
    return promise2;
  };

  // then的语法糖，将层层抛出的错误放到这里处理
  promise.prototype.catch = function(onRejected) {
    return this.then(null, onRejected);
  };

  // 不论当前promise成功失败都会执行回调并且将值原封不动的往下传
  promise.prototype.finally = function(cb) {
    let P = this.constructor;
    return this.then(
      value => {
        P.resolve(cb()).then(() => value);
      },
      error => {
        P.resolve(cb()).then(() => {
          throw error;
        });
      }
    );
  };
}
```

附带Promise的常用方法实现

```
// promise.resolve方法
promise.resolve = function(val) {
  // 如果是promise对象，直接返回
  if (val instanceof promise) return val;

  return new promise((resolve, reject) => {
    // 如果是thenable对象
    if (val && val.then && typeof val.then === "function") {
      val.then(resolve, reject);
    } else {
      resolve(val);
    }
  });
};
```
```
// promise.reject方法
promise.reject = function(err) {
  return new promise((resolve, reject) => {
    reject(err);
  });
};
```
```
// promise.all方法
promise.all = function(promises) {
  return new promise((resolve, reject) => {
    let result = []; // 结果
    let index = 0;
    let len = promises.length;
    if (len === 0) {
      resolve(result);
      return;
    }

    for (let i = 0; i < len; i++) {
      promise.resolve(promises[i]).then(
        data => {
          result[i] = data;
          index++;
          if (index === len) resolve(result);
        },
        err => {
          reject(err);
        }
      );
    }
  });
};
```
```
// promise.race方法
promise.race = function(promises) {
  return new promise((resolve, reject) => {
    let len = promises.length;
    if (len === 0) return; // 这里为什么要返回一个pending状态的promise

    for (let i = 0; i < len; i++) {
      promise.resolve(promises[i]).then(
        data => {
          resolve(data);
          return;
        },
        err => {
          reject(err);
          return;
        }
      );
    }
  });
};
```

## 一个特殊的例子

通过上面手写`Promise`，我们了解清楚了`Promise`是如何工作的，接下来我们看下面这个例子会输出什么
```
let p =  Promise.resolve(1).then(() => {
  alert(1);

  Promise.resolve(2).then(() => {
    alert(2);
  }).then(() => {
    alert(3);
  }).then(() => {
    alert(4);
  })

  alert(5);
}).then(function (result) {
  alert(6);
}).then(function (result) {
  alert(7);
});
```

答案是`1, 5, 2, 6, 3, 7, 4`，出乎意料对吗，我们试着分析一下，一开始定义了一个`Promise`，它的状态为`fulfilled`，所以第一个`then`的回调会放入微任务队列中，后面的`then`由于依赖前面的`then`的返回结果，所以状态仍为`pending`，所以不加入微任务队列

执行栈为空后，从微任务队列取出第一个`then`的回调，执行出结果`1`，接着又碰到`Promise`，和上面一样，这时又把第一个`then`的回调放入微任务队列，后面的不加入，接着输出`5`，等到执行完当前回调后，`resolve`了一个空值，这时，外部的第二个`then`的回调被加入微任务队列，我们继续执行微任务队列，碰到内部的第一个回调，输出`2`，然后加入内部的第二个回调，继续执行微任务队列，输出`6`，如此反复，最终输出`3, 7, 4`