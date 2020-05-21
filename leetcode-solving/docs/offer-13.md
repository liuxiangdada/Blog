### 题目

地上有一个 m \* n 的方格，机器人从[0, 0]的位置进入到坐标[m - 1, n - 1]停止，每次只能往上下左右移动一格，并且满足只能在方格范围内运动，不能移动到横纵坐标数位和大于 k 的坐标，求机器人能到达的格子数

### 思路

这题有一个小坑，就是机器人每次只能运动一格，有一些格子的横纵数位和虽然满足但机器人不会飞，无法到达，我们拆解一下题目，首先要实现一个求数位和的函数

```
// 递归简版
function getBit(num) {
  if(num < 10) return num;
  return num % 10 + getBit(parseInt(num / 10));
}

// 另一种递归思路，通过找规律得出递推公式
function getBit(num) {
  if(num < 10) return num;
  return num % 10 === 0 ? getBit(num - 1) - 8 : getBit(num - 1) + 1;
}

// 循环版
function getBit(num) {
  if(num < 10) return num;
  let res = 0;
  while(num) {
    let mod = num % 10;
    res += mod;
    num = (num - mod) / 10;
  }
  return res;
}
```

接着我们分析题目，大致设想一下，从头遍历，如果某点数位和满足条件且其左边的点和上面的点也可达，那么该点可达

```
var movingCount = function(m, n, k) {
  // 求数位和
  function getBit(num) {
    if(num < 10) return num;
    return num % 10 + getBit(parseInt(num / 10));
  }

  // 往可达数组添加点
  function addPoint(i, j) {
    if(i < 0 || i >= m || j < 0 || j >= n || getBit(i) + getBit(j) > k || canMove.indexOf([i, j].toString()) !== -1) return;

    if(canMove.indexOf([i - 1, j].toString()) !== -1 || canMove.indexOf([i, j - 1].toString()) !== -1) {
      canMove.push([i, j].toString());
    }
  }

  let canMove = ['0,0'];
  for(let i = 0; i < m; i++) {
    for(let j = 0; j < n; j++) {
      addPoint(i, j);
    }
  }

  return canMove.length;
};
```

### 其他解法

该题是一个矩阵搜索题，遇到这种搜索我们会联想到深度优先搜索和广度优先搜索

#### 深度优先搜索

我们首先试着使用深度优先搜索求解，一般深度优先搜索使用递归，所以我们要找出递归边界和推导式

- 搜索时如果一个点超出方格范围或者已经搜索过或者横纵坐标数位和大于 k 视为深度搜索的边界，遇到边界时，我们进行剪枝，立即返回
- 递归时判断当前坐标值是否符合递归终止条件，如果符合返回 0，否则往下和往右继续递归

```
var movingCount = function(m, n, k) {
  function dfs(i, j, bitI, bitJ) {
    if(i < 0 || i >= m || j < 0 || j >= n || bitI + bitJ > k || visited[i][j]) {
      return 0;
    }

    visited[i][j] = 1;

    return 1 + dfs(i + 1, j , (i + 1) % 10 === 0 ? bitI - 8 : bitI + 1, bitJ) + dfs(i, j + 1, bitI, (j + 1) % 10 === 0 ? bitJ - 8 : bitJ + 1)
  }

  let visited = Array(m).fill(0);
  for(let i = 0; i < m; i++) visited[i] = Array(n).fill(0);

  return dfs(0, 0, 0, 0);
};
```

#### 广度优先搜索

广度优先搜索是指每次搜索的面尽可能的广，然后再根据本次的搜索结果进行下一次搜索，广度优先搜索一般使用队列实现，每次队列中存放上一次搜索到的所有可达点

```
var movingCount = function(m, n, k) {
  function getBit(num) {
    if(num < 10) return num;
    let res = 0;
    while(num) {
      let mod = num % 10;
      res += mod;
      num = (num -mod) / 10;
    }
    return res;
  }

  let cnt = 1;
  let queue = [[0, 0]];
  let dx = [0, 1];
  let dy = [1, 0];
  let visited = Array(m).fill(0);
  for(let i = 0; i < m; i++) visited[i] = Array(n).fill(0);
  visited[0][0] = 1;

  while(queue.length > 0) {
    let [x, y] = queue.pop();
    let posX, posY;
    for(let i = 0; i < 2; i++) {
      posX = x + dx[i];
      posY = y + dy[i];

      if(posX < 0 || posX >= m || posY < 0 || posY >= n || getBit(posX) + getBit(posY) > k || visited[posX][posY]) {
        continue;
      }

      queue.push([posX, posY]);
      visited[posX][posY] = 1;
      cnt++;
    }
  }

  return cnt;
};
```

#### 递推法

通过前面我自己的解法可以稍微改进一下，我们知道一个点可达可解的条件是可解的前提下可达，一个点可达那么其左边的点和上面的点一定可达，这样我们可以稍微调整一下

```
var movingCount = function(m, n, k) {
  function getBit(num) {
    if(num < 10) return num;
    let res = 0;
    while(num) {
      let mod = num % 10;
      res += mod;
      num = (num -mod) / 10;
    }
    return res;
  }

  let cnt = 1;
  let visited = Array(m).fill(0);
  for(let i = 0; i < m; i++) visited[i] = Array(n).fill(0);
  visited[0][0] = 1;

  for(let i = 0; i < m; i++) {
    for(let j = 0; j < n; j++) {
      if((i === 0 && j === 0) || getBit(i) + getBit(j) > k) continue;

      if(i - 1 >= 0 && j >= 0) visited[i][j] |= visited[i - 1][j];
      if(i >= 0 && j - 1 >= 0) visited[i][j] |= visited[i][j - 1];

      cnt += visited[i][j];
    }
  }

  return cnt;
};
```
