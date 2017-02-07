---
title: 排序算法和其他
date: 2017-02-05
categories: 笔记
author: Jimmy Q
permalink: sort-and-others
---

## 归并排序

据说多数js引擎中数组的sort()方法使用的就是归并排序：

```javascript
function mergeSort(nums) {
    var length = nums.length;

    if (length < 2) {
        return nums;
    }

    var middle = Math.floor(length);
    var left = nums.slice(0, middle);
    var right  = nums.slice(middle);

    return merge(mergeSort(left), mergeSort(right));
}

function merge(left, right) {
    var result = [];
    while (left.length && right.length) {
        if (left[0] < right[0]) {
            result.push(left.shift());
        } else {
            result.push(right.shift());
        }
    }

    return result.concat(left, right);
}

```

### 计算复杂度

归并排序的思想是 Divide and Conquer，为了简便的分析复杂度，下面使用 pseudo code　描述解决方案，伪代码每一行代表一次计算：

```
    split each element into partitions of size 1 or 0
        recursively merge adjancent partitions
        for i = leftPartStartIndex to rightPartLastIndex inclusive
            if leftPartHeadValue <= rightPartHeadValue
            copy leftPartHeadValue
            else: copy rightPartHeadValue
        copy elements back to original array
```

根据伪代码，每次 merge 有 2n + 1 次计算，为了计算方便，我们设计算次数为 3n；

首先用递归，将数组分解成长度为1或0的数组，一共要分 log(n) 次，递归树一共有 log(n) + 1 层；

设 j=0,1,2,...,log(n)，第 j 层有 2^j 个数组，每个数组的长度为 n/(2^j)层；

因此可以得到每一层需要的计算是 2^j * 3(n/(2^j)) = 3n;

一共有 log(n) 层，因此总共的计算量为 3n(log(n) + 1);


## 快速排序



