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

# 扩展思考

## 二分查找还有改进的空间吗?

可以细微的评定下上面非递归版本的比较次数,假设以7个有序数列为例，图中能够清晰的看到查找一个元素成功或者失败需要比较次数。

![binary](/images/binarysearch/example1.png)



先看成功的情况：分别为所有次数相加除以元素个数 :(2+3+4+4+5+5+6)29/7≈4+

再看失败的情况:上图中虚线框中为失败的比较次数:(3+4+4+5+4+5+5+6)36/8=4.5

故而可以得出平均查找长度均大致为O(1.5·㏒₂n)。

可以发现向左和向右分支前的比较次数不想等。向左分支前只需要比较一次，而向右分支则需要比较两次；由此可以发现虽然递归深度是均衡的，但是转向成本不均衡。

是不是可以通过调整递归深度的不均衡，对转向成本的不均衡进行补偿呢。

### Fibonacci查找

先看一段代码

```java
Integer fibSearch(Integer[] arrs, Integer val, Integer lo, Integer hi) {
    	Fib fibs = Fib.create(hi - lo) //创建一个Fib数列 logn(hi-lo)
        while(hi >= lo){
            while( hi - lo < fib.get() ) fib.prev();//通过向前的顺序查找确定轴点
            //Integer mid = (hi + lo) / 2;
            Integer mid = lo + fib.get() - 1 //黄金比例切分
            if (val > arrs[mid]) lo = mid + 1;
            else if (val < arrs[mid]) hi = mid - 1;
            else return mid;
        }
        return -1;
}
```

还是以7个有序数列为例，看看FIb查找元素的比较次数

![binary](/images/binarysearch/example2.png)

平均成功查找比较次数=(2+3+4+4+5+5+5) / 7 = 28 / 7 = 4.00

平均失败查找比较次数=(4+5+4+4+5+4+5+4) / 8 = 35 / 8 = 4.38

跟上面的比较次数想比是不是有所优化。

普通二分查找和FIb数列查找的不同其实就是选择轴点， 二分对应轴点=0.5;FIb查找对应抽点=0.6180399...所谓的黄金分割。

### 二分查找再优化

既然之前二分查找版本左右分支的转向代价是不平衡的，那么有办法能够做成平衡的吗?

比如只做一次比较，所有分支只有2个方向，而不是3个方向。

```java
Integer binarySearchV1(Integer[] arrs, Integer val, Integer lo, Integer hi) {
        while(1 < hi - lo ){//查找宽度缩短至1时，算法才会终止
            Integer mid = ((hi - lo) >> 2) + lo;//还是以中点为轴点
            val < arrs[mid] ? hi = mi : lo = mi//每次比较都只有2个方向
        }
        return (val == arrs[lo]) ? lo : -1;
}
```

但是相对于之前的版本，由于无论如何都需要比较完所有转向，所以最好的情况下更坏（例如第一次就能比配到需要查找的元素，之前版本是O(1)， 而优化的版本需要O(logn)），但是更坏的情况变好(之前的版本需要 1.5*logn，而优化版本只需要logn)。

