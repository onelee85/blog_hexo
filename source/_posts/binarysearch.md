title: 二分查找真的不简单
author: James
tags:
  - 二分查找
categories:
  - 算法
date: 2015-04-09 15:19:00
---

# 简介

二分查找法： 在一个有序数组中，取数组中的中间值和要找的值进行比较，当要查找的值大于中间值，则在右边的区间继续取一个中间值和要比较的数进行比较。当找查找的值小于中间值时则反之，直至最后要查找的值和中间值相同，则说明找到该值。

一般而言，对于包含n个元素的有序列表，用二分查找最多需要`㏒₂n`步，而简单查找最多需要n步。

# 问题 

编写二分查找法的代码看似简单，其实是很容易出错的，你必须注意各种边界条件的处理，例如数组是空的，或者数组只有一个元素等等。别说你会出错，很多正规出版的算法书在讲解二分查找法时，他们给出的代码也是有错误的，据统计在美国出版的二十多本不同的讲算法的书中，只有五本对二分查找的代码实现是正确的。



# show code

## 递归二分查找

```java
//递归二分查找
Integer binarySearch(Integer[] arrs, Integer val, Integer lo, Integer hi) {
        if(lo > hi) return -1;
        Integer mid = (hi + lo) / 2;
        if (val > arrs[mid]) return binarySearch(arrs, val, mid + 1, hi);
        else if (val < arrs[mid]) return binarySearch(arrs, val, lo, mid - 1);
        return mid;
}
```

## 非递归

```java
Integer binarySearchV1(Integer[] arrs, Integer val, Integer lo, Integer hi) {
        while(hi >= lo){
            //low + high 是会溢出的。只要这个数组我们开的足够大
            //Integer mid = (hi + lo) / 2;
            Integer mid = ((hi - lo) >> 2) + lo;
            if (val > arrs[mid]) lo = mid + 1;
            else if (val < arrs[mid]) hi = mid - 1;
            else return mid;
        }
        return -1;
}
```

# 易错点

- *死循环*: 在做的是整数运算，整除2了之后，对于奇数和偶数的行为还不一样，很有可能有些情况下我们并没有减小取值范围，而形成死循环
- 退出条件。到底什么时候我们才觉得我们找不到呢
- 差1错误。我们的左端点应该是当前可能区间的最小范围，那么右端点是最大范围呢，还是最大范围+1呢。我们取了中间值之后，在缩小区间时，有没有保持左右端点的这个假设的一致性呢？