### 题目

给出一个n * n的矩阵，求其顺时针旋转90度后的矩阵，要求原地交换，不使用额外内存

### 思路

1.首先按照正常的思维方式求解，我们首先拷贝一份矩阵，然后按照旋转90度后的下标进行遍历，对原矩阵重新赋值

```
var rotate = function(matrix) {
  let len = matrix.length;
  // 深拷贝，防止篡改原数组
  let matrix2 = Array(len).fill(0);
  for (let i = 0; i < len; i++) {
    matrix2[i] = matrix[i].slice();
  }
  let pos;
  for (let i = 0; i < len; i++) {
    pos = len - 1;
    for (let j = 0; j < len; j++) {
      matrix[i][j] = matrix2[pos][i];
      pos--;
    }
  }
};
```
成功AC，但是使用了n*n的额外空间，需要重新求解

2.我们对矩阵进行找规律，发现n为偶数时，需要从外到内旋转n/2圈，为奇数时需要旋转(n - 1)/2圈，每次旋转n个点，需要交换n-1次

```
/**
 * 原地交换
 * 内层循环遍历，外层控制边界
 * (i, j)        ->        (j, n - i - 1)
 *
 *
 * (n - j - 1, i)    <-    (n - i - 1, n - j - 1)
 */
var rotate = function(matrix) {
  let n = matrix.length;
  // n为偶数则需要旋转n/2圈，为奇数则需要旋转(n - 1)/2圈
  for (let i = 0; i < n / 2; i++) {
    // 每次需要旋转一圈的n - 1个点
    for (let j = i; j < n - i - 1; j++) {
      let t = matrix[i][j];
      matrix[i][j] = matrix[n - j - 1][i];
      matrix[n - j - 1][i] = matrix[n - i - 1][n - j - 1];
      matrix[n - i - 1][n - j - 1] = matrix[j][n - i - 1];
      matrix[j][n - i - 1] = t;
    }
  }
};
```

3.矩阵有这样的性质，其转置矩阵等于其顺时针旋转90后再倒序排序，我们可以求逆运算，先求出其转置矩阵，即根据对角线交换元素，然后对每一行进行倒序

```
// 先对矩阵进行转置，然后倒序
var rotate = function(matrix) {
  let n = matrix.length;
  for (let i = 0; i < n; i++) {
    // 取j的下标为i表示每次外循环结束后以i为坐标值的位置都已经被交换
    for (let j = i; j < n; j++) {
      [matrix[i][j], matrix[j][i]] = [matrix[j][i], matrix[i][j]];
    }
  }

  matrix.forEach(item => item.reverse());
};
```
