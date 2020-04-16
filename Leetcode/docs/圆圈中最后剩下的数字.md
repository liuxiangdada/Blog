### 题目

0,1,...,n-1这n个数字排成一个圆圈，从数字0开始，每次从这个圆圈里删除第m个数字。求出这个圆圈里剩下的最后一个数字，例如，0、1、2、3、4这5个数字组成一个圆圈，从数字0开始每次删除第3个数字，则删除的前4个数字依次是2、0、4、1，因此最后剩下的数字是3。
(1 <= n < 10^5；1 <= m <= 10^6)

### 思路

1.这个问题我看到后首先想到的是循环链表解法，首先构建一个长度为n的循环链表，然后以m间隔剔除元素，最后返回链表中的唯一元素

```
var lastRemaining = function(n, m) {
  // 链表
  function list(val) {
    this.val = val;
    this.next = null;
  }

  let node, head;
  for (let i = 0; i < n; i++) {
    if (!node) {
      head = node = new list(i);
    } else {
      node = node.next = new list(i);
    }
  }

  node.next = head;

  let prev = node, curr = head;
  while (curr.next) {
    if (curr.next === curr) break;
    for (let i = 1; i < m; i++) {
      prev = curr;
      curr = curr.next;
    }
    let next = curr.next;

    // 处理第m个数字
    prev.next = next;
    curr.next = null;
    curr = next;
  }

  return curr.val;
};
```

**_测试通过后提交，发现超时了，看了评论，发现题目对n和m的取值范围有所要求：而这个解法的时间复杂度为O(nm)显然在oj测试下会超时_**

2.换一个思路，其实这是一个约瑟夫环问题，通过百度了解到其实是有数学递推公式的，我们简单递推一下

当人数n为1时，结果显然为0，得到F(1) = 0
当人数为2时，假设m = 3， 则F(2) =  3 % 2 = 1
当人数为3时，假设m = 3， 则F(3) =  (1 + 3) % 3 = 1
当人数为n时，假设m = 3， 则F(n) =  (F(n - 1) + 3) % n 
去掉m假设，则得到F(n) =  (F(n - 1) +m) % n 

所以递推公式为
F(1) = 0  (n = 1)
F(n) = (F(n - 1) +m) % n  (n > 1)

```
// 递归版
var lastRemaining = function(n, m) {
  if(n === 1) return 0;
  return (lastRemaining(n - 1) + m) % n;
};
```

```
// 循环版
var lastRemaining = function(n, m) {
  let ans = 0;
  for(let i = 2; i <= n; i++) {
    ans = (ans + m) % i;
  }
  return ans;
};
```

