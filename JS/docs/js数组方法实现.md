### 数组的map方法

```
Array.prototype.map = function(callbackFn, thisArg) {
  // 处理调用异常
  if (this === null || this === undefined) {
    throw new TypeError("Cannot read property 'map' of null or undefined");
  }

  // 处理回调函数异常
  if (Object.prototype.toString.call(callbackFn) !== "[object Function]") {
    throw new TypeError(callbackFn + " isn't a function");
  }

  let O = Object(this);
  let len = O.length >>> 0; // 确保len为数字且为整数
  let T = thisArg;
  let res = [];

  for (let i = 0; i < len; i++) {
    if (i in O) {
      let element = O[i];
      res[i] = callbackFn.call(T, O[i], i, O);
    }
  }

  return res;
}

```

### 数组的reduce方法

```
Array.prototype.reduce = function(callbackFn, current) {
  // 处理调用异常
  if (this === null || this === undefined) {
    throw new TypeError("Can not read property 'reduce' of null or undefined");
  }

  let O = Object(this);
  let len = O.length >>> 0; // 确保len为数字且为整数

  return innerArrayReduce(callbackFn, current, O, len, arguments.length);

  function innerArrayReduce(callbackFn, current, O, len, argumentLen) {
    // 处理回调函数异常
    if (Object.prototype.toString.call(callbackFn) !== "[object Function]") {
      throw new TypeError(callbackFn + " isn't a function");
    }

    let i = 0;

    find_inital: if (argumentLen < 2) {
      for (; i < len; i++) {
        if (i in O) {
          current = O[i];
          break find_inital;
        }
      }

      throw new TypeError("array is empty");
    }

    for (; i < len; i++) {
      if (i in O) {
        current = callbackFn.call(undefined, current, O[i], i, O);
      }
    }

    return current;
  }
}
```

### 数组的pop方法

```
Array.prototype.pop = function() {
  // 处理调用异常
  if (this === null || this === undefined) {
    throw new TypeError("Can not read property 'pop' of null or undefined");
  }

  let O = Object(this);
  let len = O.length >>> 0;

  // 数组为空的情况
  if (len === 0) {
    O.length = 0;
    return undefined;
  }

  len--;
  const element = O[len];
  delete O[len];
  O.length = len;
  return element;
}
```

### 数组的push方法

```
Array.prototype.push = function(...items) {
  // 处理调用异常
  if (this === null || this === undefined) {
    throw new TypeError("Can not read property 'push' of null or undefined");
  }

  let O = Object(this);
  let len = O.length >>> 0;
  let argCount = items.length;

  // 判断数组是否超出个数限制
  if (len + argCount > Math.pow(2, 53) - 1) {
    throw new TypeError("the length of the array is overSize");
  }

  for (let i = 0; i < argCount; i++) {
    O[len + i] = items[i];
    len++;
  }

  O.length = len;
  return len;
}
```

### 数组的filter方法

```
Array.prototype.filter = function(callbackFn, thisArg) {
  // 处理调用异常
  if (this === null || this === undefined) {
    throw new TypeError("Can not read property 'filter' of null or undefined");
  }

  // 处理回调函数异常
  if (Object.prototype.toString.call(callbackFn) !== "[object Function]") {
    throw new TypeError(callbackFn + " isn't a function");
  }

  let O = Object(this);
  let len = O.length >>> 0; // 确保len为数字且为整数
  let T = thisArg;
  let res = [];
  let resPos = 0;

  for (let i = 0; i < len; i++) {
    if (i in O) {
      let element = O[i];
      if (callbackFn.call(T, O[i], i, O)) {
        res[resPos++] = element;
      }
    }
  }

  return res;
}
```

### 数组的splice方法

```
Array.prototype.splice = function(startIndex, deleteCount, ...addItems) {
  // 处理调用异常
  if (this === null || this === undefined) {
    throw new TypeError("Can not read property 'splice' of null or undefined");
  }

  let array = Object(this);
  let len = array.length >>> 0;
  let deleteArr = new Array(deleteCount);
  let argumentsLen = arguments.length >>> 0;

  function computeStartIndex(startIndex, len) {
    startIndex = +startIndex;

    if (isNaN(startIndex)) {
      throw new TypeError("please provide a valid argument for startIndex");
    }

    if (startIndex < 0) {
      return startIndex + len > 0 ? startIndex + len : 0;
    }

    return startIndex >= len ? len : startIndex;
  }

  function computeDeleteCount(startIndex, deleteCount, len, argumentsLen) {
    if (argumentsLen === 1) {
      return len - startIndex;
    }

    deleteCount = +deleteCount;

    if (isNaN(deleteCount)) {
      throw new TypeError("please provide a valid number for delete");
    }

    if (deleteCount < 0) {
      return 0;
    }

    if (deleteCount > len - startIndex) {
      return len - startIndex;
    }

    return deleteCount;
  }

  function sliceDeleteItems(array, startIndex, deleteCount, deleteArr) {
    for (let i = 0; i < deleteCount; i++) {
      let index = i + startIndex;
      if (index in array) {
        deleteArr[i] = array[index];
      }
    }
  }

  function movePostItems(array, startIndex, deleteCount, len, addItems) {
    let addLen = addItems.length;

    // 删除元素个数和增加元素个数相等
    if (deleteCount === addLen) {
      return;
    }
    // 删除元素个数大于增加元素个数
    else if (deleteCount > addLen) {
      for (let i = startIndex + deleteCount; i < len; i++) {
        let fromIndex = i;

        let toIndex = i - (deleteCount - addLen);

        if (fromIndex in array) {
          array[toIndex] = array[fromIndex];
        }
      }

      // 删除移动后的冗余元素
      for (let i = len - 1; i >= len + addLen - deleteCount; i--) {
        delete array[i];
      }
    }
    // 删除元素个数小于增加元素个数
    else if (deleteCount < addLen) {
      for (let i = len - 1; i >= startIndex + deleteCount; i--) {
        let fromIndex = i;

        let toIndex = i + (addLen - deleteCount);

        if (fromIndex in array) {
          array[toIndex] = array[fromIndex];
        }
      }
    }
  }

  // 处理参数，非数值情况、边界情况、负值情况
  startIndex = computeStartIndex(startIndex, len);
  deleteCount = computeDeleteCount(startIndex, deleteCount, len, argumentsLen);

  // 处理密封对象和冻结对象
  if (Object.isSealed(array) && deleteCount > addLen) {
    throw new TypeError("this is a sealed object");
  }
  if (Object.isFrozen(array) && (deleteCount > 0 || addLen > 0)) {
    throw new TypeError("this is a frozen object");
  }

  // 拷贝删除的元素
  sliceDeleteItems(array, startIndex, deleteCount, deleteArr);

  // 移动删除元素后面的元素
  movePostItems(array, startIndex, deleteCount, len, addItems);

  // 插入新元素
  for (let i = 0; i < addItems.length; i++) {
    array[startIndex + i] = addItems[i];
  }

  array.length = len + addItems.length - deleteCount;

  return deleteArr;
}
```

###  数组的sort方法

#### 大致思路
<10时使用插入排序，>10时使用快速排序（Chrome20以前，以后的版本已经改用TimSort）

#### 基准选取
其中10-1000时基准为中间位置的那个值，>1000时除去首尾的每200-215个间隔取一个点顺序排序取中间值，把选好的基准跟首尾元素排序，取中点为最终基准

#### 快排

1. 将基准置于未排序数列的首位（与首位交换位置）
2. 首先从左边开始遍历未比较数列，找到比基准小的数，就更新小区间的范围(与左边游标交换位置)并移动基准位置
3. 如果比基准大，从右边判断比基准小的数(期间如果当前游标与右边游标相会则说明分区结束)，对找到的这个数再次判断，如果等于基准，则更新相等区间的范围，如果小于基准，则更新小区间范围

#### 遍历示意图

from---------------lowEnd-------------i-----------------highStart------------to

|-------小于基准 ------|---等于基准---|------- 未比较--------|--- 大于基准---|

#### V8优化

V8引擎单独对递归做了优化，针对筛出的两个区间，对小区间使用递归，大区间则使用循环
如果前面的区间大于后面，则对后面的区间递归，这时hightStart-to的区间已被排序，未排序部分为from-lowEnd，反之则反

from---------------------------lowEnd/highStart----------------to
|----------------未比较--------------|------------已比较-----------|



```
const innerArraySort = function(array, len, compareFn) {
  // 处理回调
  if (Object.prototype.toString.call(compareFn) !== "[object Function]") {
    compareFn = function(x, y) {
      if (x === y) return 0;
      x = x.toString();
      y = y.toString();
      if (x === y) return 0;
      else return x < y ? -1 : 1;
    };
  }

  // 插入排序
  const insertSort = function(arr, from, to) {
    for (let i = from + 1; i < to; i++) {
      let element = arr[i]; // 当前未排序数
      let j = i - 1;
      for (; j >= from; j--) {
        let order = compareFn(element, arr[j]);
        if (order < 0) {
          arr[j + 1] = arr[j];
        } else {
          break;
        }
      }
      arr[j + 1] = element;
    }
  };

  // 找到哨兵元素
  const getThirdIndex = function(arr, from, to) {
    let tmpArr = [];

    let increment = 200 + ((to - from) & 15); // 把地增量控制在200-215之间

    let j = 0;

    from += 1;
    to -= 1;

    for (let i = from; i < to; i += increment) {
      tmpArr[j] = [i, arr[i]];
      j++;
    }

    // 排序临时数组
    tmpArr.sort(function(a, b) {
      return compareFn(a[1], b[1]);
    });

    let thirdIndex = tmpArr[tmpArr.length >> 1][0];

    return thirdIndex;
  };

  // 快排
  const _sort = function(a, b, c) {
    let arr = [a, b, c];
    insertSort(arr, 0, 3);
    return arr;
  };

  const quickSort = function(arr, from, to) {
    let thirdIndex = 0; // 哨兵元素

    while (true) {
      if (to - from <= 10) {
        insertSort(arr, from, to);
        return;
      } else if (to - from > 1000) {
        thirdIndex = getThirdIndex(arr, from, to);
      } else {
        // 小于1000取中点作为哨兵元素
        thirdIndex = from + ((to - from) >> 2);
      }

      // 再次判断哨兵元素是否合适
      [arr[from], arr[thirdIndex], arr[to - 1]] = _sort(
        arr[from],
        arr[thirdIndex],
        arr[to - 1]
      );

      let lowEnd = from + 1; // 基准所在位置
      let highStart = to - 1; // 倒序查找标记

      let pivot = arr[thirdIndex];
      arr[thirdIndex] = arr[lowEnd];
      arr[lowEnd] = pivot;

      for (let i = lowEnd + 1; i < highStart; i++) {
        let element = arr[i];
        let order = compareFn(element, pivot);

        if (order < 0) {
          arr[i] = arr[lowEnd];
          arr[lowEnd] = element;
          lowEnd++;
        } else if (order > 0) {
          do {
            highStart--;
            if (highStart === i) break;

            order = compareFn(arr[highStart], pivot);
          } while (order > 0);

          arr[i] = arr[highStart];
          arr[highStart] = element;

          if (order < 0) {
            element = arr[i];
            arr[i] = arr[lowEnd];
            arr[lowEnd] = element;
            lowEnd++;
          }
        }
      }

      if (lowEnd - from > to - highStart) {
        quickSort(arr, highStart, to);
        to = lowEnd;
      } else if (lowEnd - from <= to - highStart) {
        quickSort(arr, from, lowEnd);
        from = highStart;
      }
    }
  };

  quickSort(array, 0, len);
};
Array.prototype.sort = function(compareFn) {
  // 处理调用异常
  if (this === null || this === undefined) {
    throw new TypeError("Can not read property 'sort' of null or undefined");
  }

  let array = Object(this);
  let len = array.length >>> 0;

  return innerArraySort(array, len, compareFn);
}
```
