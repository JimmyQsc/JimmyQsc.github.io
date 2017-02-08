---
title: 排序算法和其他
date: 2017-02-05
categories: 笔记
author: Jimmy Q
permalink: sort-and-others
---

复习一下算法知识，从最基础的开始。一个人应该首先是软件工程师然后才是前端工程师。

## 归并排序

据说多数js引擎中数组的sort()方法使用的就是归并排序，用javascript实现一下：

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

遵循算法分析的三个准则：
    * 分析最坏情况。对于输入大小n，我们的分析结果是个上限，这样数学上最简单。
    * 不关注常因数和低阶的术语。也就是不关心具体写了几行代码，使用了什么语言。（优化的细节）
    * 渐进分析，关注极大的输入量，随着输入量的增大，常因数的影响会越来越小。

快的算法就是随着输入量增大，最坏运行时间增长速度慢的算法。

归并排序的思想是 Divide and Conquer，为了简便的分析复杂度，首先忽略语言层面的东西，下面使用 pseudo code　描述解决方案，伪代码每一行代表一次计算：

<code style="font-family: monospace, monospace; background: #fff">
    split each element into partitions of size 1 or 0
        recursively merge adjancent partitions
        for i = leftPartStartIndex to rightPartLastIndex inclusive
            if leftPartHeadValue <= rightPartHeadValue
            copy leftPartHeadValue
            else: copy rightPartHeadValue
        copy elements back to original array
</code>

根据伪代码，每次 merge 有 2n + 1 次计算，为了计算方便，我们设计算次数为 3n；

首先用递归，将数组分解成长度为1或0的数组，一共要分 log(n) 次，递归树一共有 log(n) + 1 层；

设 j=0,1,2,...,log(n)，第 j 层有 2^j 个数组，每个数组的长度为 n/(2^j)层；

因此可以得到每一层需要的计算是 2^j * 3(n/(2^j)) = 3n;

一共有 log(n) 层，因此总共的计算量为 3n(log(n) + 1);

然后我们用big-OH notation得出算法复杂度为 O(nlog(n));


## 快速排序





