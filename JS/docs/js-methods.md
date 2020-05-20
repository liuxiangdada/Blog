### 模拟bind的实现

#### 函数版

```
function bindOperator(thisArg, fn, ...args) {
  // 判断是否是函数调用
  if (Object.prototype.toString.call(fn) !== "[object Function]") {
    throw new TypeError(
      "Function.prototype.bind - what is trying to be bound is not callable"
    );
  }

  let A = function() {
    // 判断调用新函数的对象是new的对象还是普通函数调用，如果是普通调用，this绑定为传入对象，否则为new出来的对象
    return fn.apply(
      this instanceof fn ? this : thisArg,
      args.concat(Array.prototype.slice.call(arguments))
    );
  };

  A.prototype = Object.create(fn.prototype); // 如果是构造函数调用，修正原型对象

  return A;
}
```

#### 原型版

```
Function.prototype.bind = function(thisArg, ...args) {
  // 判断是否是函数调用
  if (Object.prototype.toString.call(this) !== "[object Function]") {
    throw new TypeError(
      "Function.prototype.bind - what is trying to be bound is not callable"
    );
  }

  let self = this; // 保存当前调用函数
  let A = function() {
    return self.apply(this instanceof self ? this : thisArg, args.concat([...arguments]));
  }

  A.prototype = Object.create(self.prototype);

  return A;
}
```

### 模拟call的实现

#### 函数版

```
function callOperator(thisArg, fn, ...args) {
  let context = thisArg || window;

  let f = Symbol("f");
  context.f = fn; // 保存当前调用函数

  let result = context.f(...args);
  delete context.f;

  return result;
}
```

#### 原型版

```
Function.prototype.call = function(thisArg, ...args) {
  let context = thisArg || window;
  let fn = Symbol("fn");
  context.fn = this;

  let result = context.fn(...args);

  delete context.fn;

  return result;
};
```
#### ES5版

```
Function.prototype.call = function(thisArg) {
  var context = thisArg || window;
  var fn = '__callFnPro'; // 设置一个不常见不可仿造的属性名
  context.fn = this; // 保存当前函数

  var args = [];
  for(var i  = 1; i < arguments.length; i++) {
    args.push('arguments[' + i + ']');
  }

  var result = eval('context.fn(' + args + ')');

  delete context.fn;

  return result;
};
```

### 模拟apply的实现

#### 函数版

```
function applyOperator(thisArg, fn, args) {
  let context = thisArg || window;

  let f = Symbol("f");
  context.f = fn; // 保存当前调用函数

  let result = context.f(...args);
  delete context.f;

  return result;
}
```

#### 原型版

```
Function.prototype.apply = function(thisArg, args = []) {
  let context = thisArg || window;
  let fn = Symbol("fn");
  context.fn = this;

  let result = context.fn(...args);

  delete context.fn;

  return result;
};
```

### 模拟new的实现

```
function newOperator(fn, ...args) {
  // 判断是否是函数调用
  if (Object.prototype.toString.call(fn) !== "[object Function]") {
    throw new TypeError(fn + " is not a constructor");
  }

  // 新建一个对象，以调用函数的原型对象为原型
  let obj = Object.create(fn.prototype);

  // 执行这个函数中的代码，为新对象增加属性
  let res = fn.apply(obj, args);

  // 引用数据类型
  let isObject = typeof res === "object" && res !== null;
  let isFunction = typeof res === "function";

  return isObject || isFunction ? res : obj;
}
```

### 深拷贝，可以无限极的拷贝嵌套对象

#### 需要解决三个问题

1. 循环引用
2. 无法拷贝特殊对象：RegExp、Date等
3. 无法拷贝函数

```
function deepCopy(target, map = new WeakMap()) {
  // 判断对象
  function isObject(val) {
    return (
      (typeof val === "object" && val !== null) || typeof val === "function"
    );
  }

  // 处理正则对象，如果直接传入正则作为参数，会返回原正则的拷贝
  function handleRegExpObject(reg) {
    let { source, flags } = reg;
    return new reg.constructor(reg, flags);
  }

  // 处理函数对象
  function handleFunctionObject(fn) {
    // 处理箭头函数，箭头函数无原型对象，直接返回
    if (fn.prototype === undefined) return fn;

    /**
     * 正则模式匹配
     * (?:pattern)，非获取匹配，匹配pattern但不获取结果，不进行储存
     * (?=pattern)，非获取匹配，正向肯定预查，在pattern处开始匹配字符串
     * (?!pattern)，非获取匹配，正向否定预查，在不匹配pattern处开始匹配字符串
     * (?<=pattern)，非获取匹配，反向肯定预查，在pattern处开始向前匹配字符串
     * (?<!pattern)，非获取匹配，反向否定预查，在不匹配pattern处开始向前匹配字符串
     * m修饰符为多行匹配
     */
    // 参数匹配正则
    const paramReg = /(?<=\().+(?=\)\s+{)/;
    // 函数体匹配正则
    const bodyReg = /(?<={)(.|\n)+(?=\s+})/m;

    let fnStr = fn.toString();

    let paramArr = paramReg.exec(fnStr)[0].split(",");
    let body = bodyReg.exec(fnStr)[0];

    // 无函数体
    if (!body) return null;

    if (paramArr) {
      return new Function(...paramArr, body);
    } else {
      return new Function(body);
    }
  }

  // 处理不可遍历对象类型
  function handleNoneIteratorObject(target, type) {
    let Ctor = target.constructor; // 获取对象的构造器

    switch (type) {
      case numberTag:
        return new Object(Number.prototype.valueOf.call(target));
      case stringTag:
        return new Object(String.prototype.valueOf.call(target));
      case boolTag:
        return new Object(Boolean.prototype.valueOf.call(target));
      case symbolTag:
        return new Object(Symbol.prototype.valueOf.call(target));
      case errorTag:
      case dateTag:
        return new Ctor(target);
      case regExpTag:
        return handleRegExpObject(target);
      case functionTag:
        return handleFunctionObject(target);
      default:
        return new Ctor(target);
    }
  }

  // 对象类型
  let type = Object.prototype.toString.call(target);

  // 可遍历对象
  let canIterater = {
    "[object Array]": 1,
    "[object Map]": 1,
    "[object Set]": 1,
    "[object Object]": 1,
    "[object Arguments]": 1
  };

  // 单独处理的对象类型
  let mapTag = "[object Map]";
  let setTag = "[object Set]";
  // 不可遍历对象类型
  let regExpTag = "[object RegExp]";
  let dateTag = "[object Date]";
  let errorTag = "[object Error]";
  let numberTag = "[object Number]";
  let boolTag = "[object Boolean]";
  let stringTag = "[object String]";
  let symbolTag = "[object Symbol]";
  // 函数对象比较特殊，单独处理
  let functionTag = "[object Function]";

  // 不是对象，原路返回
  if (!isObject(target)) return target;

  let clone;
  // 处理不可遍历对象
  if (!canIterater(type)) {
    return handleNoneIteratorObject(target, type);
  } else {
    // 保证对象的原型不丢失
    clone = new target.prototype(target);
  }

  // 处理循环引用
  if (map.get(target)) {
    return target;
  }
  // 拷贝过的对象记录到map中，后面如果再次遇到直接返回
  map.set(target, true);

  // 处理Map对象
  if (mapTag === type) {
    for (let [key, val] of target) {
      clone.set(deepCopy(key, map), deepCopy(val, map));
    }
  }

  // 处理Set对象
  if (setTag === type) {
    for (let val of target) {
      clone.add(deepCopy(val, map));
    }
  }

  // 处理其他可遍历对象
  for (let i in target) {
    if (target.hasOwnProperty(i)) {
      let element = target[i];
      clone[i] = deepCopy(element, map);
    }
  }

  return clone;
}
```
