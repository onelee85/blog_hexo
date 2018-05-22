title: 快速排序了解下
author: James
tags:
  - 快速排序
categories:
  - 算法
date: 2015-05-17 10:03:00
---

# 前言

排序算法可能是应用最广泛的算法了，它的特点是简单、适用于不同的输入数据并且在一般的应用中比其他的排序算法要快。快速排序采用的也是分治思想，但是与归并排序不同的是快速排序是原地排序并不需要使用辅助数组能节省更多的内存资源。 

<!-- more -->

# 基本算法

快速排序是将一个数组分成两个子数组，并将两个子数组独立的排序。数组的切分与归并排序有很大的不同，快速排序是找出一个元素作为基准，然后确定这个基准元素在数组中的位置，在这个过程中使基准左边的元素都不大于它右边的元素都不小于它。快速排序就是递归的调用切分的方法来完成排序的。 

## 切分的方式

快速排序中最重要的就是步骤就是将小于等于中轴元素的放到中轴元素的左边，将大于中轴元素的放到中轴元素的右边，我们暂时把这个步骤定义为切分。而剩下的步骤就是进行递归而已，递归的边界条件为数组的元素个数小于等于1。以首元素作为中轴，看看常见的切分方式。 



```java
/* * 递归调用 * lo, hi用于跟踪递归进度 */ 
public static void sort(char[] src, int lo, int hi) { 
    if (lo >= hi) return ; 
    int splitIndex = partition(src, lo, hi); 
    //切分 
    sort(src, lo, splitIndex - 1); 
    sort(src, splitIndex + 1, hi); 
}

/* * 快速排序的核心 * 用段落的第一个元素作为切分的标准 
 将小于它的都放在左部分，大于它的都放在右部分  
 从左到右找大的，从右到左找小的，做交换 
 * 最后找到最靠右的小于标准元素的元素与标准元素进行交换 
 */ 
public static int partition(char[] src, int lo, int hi) { 
    int base = src[lo]; 
    int l = lo, h = hi + 1; 
    while (true) { 
        while (src[++l] < base) 
            if (l == hi) break; 
        while (src[--h] > base) 
            if (h == lo) break; if (l < h) 
                exchange(src, l, h); else break; 
    } /* * 这里的h换成l-1都可以，表示左侧部分最右的一个元素 */ 
    exchange(src, lo, h); return h; 
}
```

# 提高性能 

在小数组排序的时候我们可以将快速排序替换为插入排序，这样会进一步提升快速排序的性能，改进后的代码像下面这样 

```java
public static void sort(char[] src, int lo, int hi) { 
    if (lo >= hi) return ; 
    if((hi - lo +1) < 10) {
        insertionSort(arr,low, high);
        return;
    }
    int splitIndex = partition(src, lo, hi); 
    //切分 
    sort(src, lo, splitIndex - 1); 
    sort(src, splitIndex + 1, hi); 
}
```

# 再提升

在实际应用中，我们很少会处理无重复元素的数组，经常会遇到含有大量重复元素的数组，而我们之前的基础排序算法对于重复的元素仍对其进行比较和大量的交换，这显然是没有必要的，在其中我们便可以发现巨大的优化潜力，在特定的情况下将线性对数级别的性能进一步改进。 

## 三向切分

之前的算法是将数组分为两个部分，小于基准元素和大于基准元素，三向切分将数组分为三个部分：小于、等于和大于基准元素。所以我们需要更多的指针跟踪（lo, lt, i, gt, hi），我们让**a[ lo ... lt - 1 ]**存储小于基准元素的元素，**a[ lt ... i - 1]**存储等于基准元素的元素，**a[ i ... gt ]**存储未访问到、未确定大小的元素，**a[ gt + 1 ... hi ]**存储大于基准元素的元素。

```java
public static void sort(char[] arr, int lo, int hi) { 
    int n = hi - lo + 1; // 当子数组的长度为 8 时，调用插入排序 
    if (n <= CUTOFF) { insertionSort(arr, lo, hi); return; } 
    // 调用三取样切分 
    int m = median3(arr, lo, lo + n / 2, hi); 
    exchange(arr, m, lo); 
    int lt = lo; int gt = hi;
    int v = arr[lo]; 
    int i = lo;
    while (i <= gt) { 
        // arr[i] < v，交换 arr[lt] & arr[i]，将 lt & i 加一 
        if (arr[i] < v) { exchange(arr, lt++, i++); } 
        // arr[i] > v，交换 arr[gt] & arr[i]，将 gt 减一 else
        if (arr[i] > v) { exchange(arr, i, gt--); } 
        // arr[i] == v，将 i 加一 
        else { 
            i++; 
        } 
    } 
    // arr[lo...lt-1] < v = arr[lt...gt] < arr[gt+1...hi] 
    sort(arr, lo, lt - 1); 
    sort(arr, gt + 1, hi); 
}

// 取 arr[i] arr[j] arr[k] 三个元素值的中间元素的下标 
private static int median3(char[] arr, int i, int j, int k) { 
    return (less(arr[i], arr[j]) ? 
            (less(arr[j], arr[k]) ? j : less(arr[i], arr[k]) ? k : i) : 					(less(arr[k], arr[j]) ? j : less(arr[k], arr[i]) ? k : i)); 
}

```



# 快速排序的优势

相对于那些**初级排序算法**（插入排序、选择排序等），快速排序的优势不用多说，平均情况的线性对数级别比平方级别在性能上要好太多。而最差情况的平方级别完全可以通过随机化避免。

相对于**归并排序**。在空间上，归并排序的劣势显露无疑，归并排序需要线性级别的空间，所以只需要对数级别的空间。在时间上，上面已经进行了详细讨论，三向切分快速排序在实际应用中的性能优于归并排序很多。

相对于**堆排序**。堆排序在时间和空间上都有非常好的效果，但是它总是跳跃式地访问元素，无法利用缓存，而快速排序总会顺序地访问元素，可以利用缓存。而缓存对于性能的提升是非常有用的。

快速排序的内循环中的指令很少，运行时间的增长数量级为~cNlgN，并且三向切分的快速排序对于实际应用中可能出现的某些分布的输入变成了线性级别，这使得其他的排序算法望而却步。