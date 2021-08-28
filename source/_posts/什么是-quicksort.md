---
layout: title
title: 什么是 quicksort
date: 2021-08-28 18:25:37
tags:
---

*`Quick`* -> `快`
*`Sort`* -> `排序`

所以 QuickSort 就是快速的排序。本章终。

哈哈哈，开个小玩笑，我们言归正传。

<!-- more -->

## 1. 快从何来

在谈快之前，我想我们要先明白排序是什么，有快就有慢，那么为什么别的是慢的？

实际上，Donald E.Knuth 已经在《TAOCP》卷三中提到了排序的思想。甚至介绍了几十种排序方式。而我们简单写个笔记，排序的本质应用：

>1. 求解“归属”问题
>2. 匹配两个或更多文件中的项目
>3. 按键值查找信息

接下来让我们着重看快速排序吧。

快速排序使用每次比较的结果来确认接下来要比较哪些 key。基本思想：

> 1. 任意取得一条记录 R(x)，将其移动到有序文件中它应当占据的最终位置（FinalPosition下面简记为 FP）该标记记录为 R(FP)。
> 2. 在确定最终位置的同时，重新安排其他记录（Record），使得 R(FP) 的左侧没有更大的记录，右侧没有更小的记录。
> 3. 再次回到第一个记录。继续下面的步骤，知道最终完成所有记录的排序。

这样，原来的排序方式简化为两个简单问题，也就是分别对 R(1)...R(FP-1) 和 R(FP+1)...R(N) 进行独立排序。

**平均时间复杂度：nlogn 且常数因子 n 小于归并排序。**

可能看完思想后觉得非常简单，但是实际上废话一堆。因为，R(x) 是第几个记录？凭什么能一下子就将其移动到它应当占据的最终位置 R(FP)？

Ok，请先不要失望，也不要做抬杠精英。让我们来实现一下。

## 2. 实现

我们先看一下关于思想里面的话题来写一个实现。

* quicksort 函数基本思想很简单，先传入一个 int 型数组，然后传入当前数组的左边界下标和右边界下标。然后该函数返回一个在正确位置的记录的下标。

```cpp
void quicksort(int A[], int left, int right)
{
    if (left < right)
    {
        /*
        此时就完成了思路中的第一条：

        任意取得一条记录 R(x)，将其移动到有序文件中它应当占据的最终位置（FinalPosition下面简记为 FP）该标记记录为 R(FP)
        
        在此段程序中，我们向 partition 函数传入了：
         1. 待排序的 int 型数组 A;
         2. 数组 A 待排序部分左边界的值 left;
         3. 数组 A 待排序部分右边界的值 right;
         
        partition 返回了一个 int 型数字，这个数字目前已经在 A 的正确位置了。
        */
        int pivot = partition(A, left, right);
        
        // 接下来，我们将待排序部分 [left, right] 转变为了 [left, pivot)
        // 先对 数组 A 的 left 到 pivot 左边的值再次进行排序
        quicksort(A, left, pivot - 1);

        // 当 quicksort(A, left, pivot - 1); 执行完成后，我们可以确认
        // 排好了 [left, pivot) 和 pivot 这两个部分的数组值。此时排好序范围为：
        // [left, pivot]
        // 接下来，我们将待排序部分只剩下了 (pivot, right]
        // 所以，A 中待排序数字
        // 左边界是 pivot + 1
        // 有边界是 right
        quicksort(A, pivot + 1, right);
    }
}

```

* 接下来是我们分治核心 partition 的实现：

```cpp
int partition(int *A, int left, int right) {
    // 我们选取数组 A 的最左边界值作为第一个被排序数字。
    int pivot = A[left];

    // 此时成立
    while (left < right)
    {
        // 先检查右边界：
        // 最右值大于 待排序值
        // 且下标 left < right
        while (left < right && A[right] >= pivot)
            // 判断右边界的前一个
            --right;

        // 当不满足前面的条件，即 right 下标对应的值小于 待排序记录的值，
        // 将 right 下标的值赋值到 left 下标。
        A[left] = A[right];

        // 查看左边界：
        // 最右值大于 待排序值
        // 且下标 left < right
        while (left < right && A[left] <= pivot)
            // 判断左边界的后一个
            left++;

        // 当不满足前面的条件，即 left 下标对应的值大于 待排序记录的值，
        // 将 left 下标的值赋值到 right 下标。
        A[right] = A[left];
    }

    // 此时的 left 已经被移动，并且现在 left 所在的下标
    // 就 pivot 这个值应该在的当前数组待排序区间的正确位置。
    A[left] = pivot;

    // 返回这个正确的下标位置。
    return left;
}

```


## 3. 总结

快排，听起来简单，实际上实现起来全是需要注意的细节点和优化点．

而且，在 CPP 中有一种方法叫 Inline(内连函数). 而我们当前的实现是递归的方式, 并不能充分的利用这一特性.

那么应该如何做到这一点呢???

