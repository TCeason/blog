---
title: 什么是heap
date: 2021-06-30 22:32:30
categories: DataStructures
tags: Heap
---

心情不佳，我们来写个堆吧。

## 堆的定义

堆我脑子里第一个浮现的是金字塔的外形。但是，抽象一下，更像是一棵树。

而在很多算法题中，有些 Top K 问题，在关键字和提示中，也经常出现 `堆` 这个字眼。

因此，简单来说，可以认为堆满足一些性质：

* 堆中某个节点的值总是不大于或不小于父节点（因此有了最大堆和最小堆的说法，父最大为最大堆，父最小为最小堆）

* 堆是一个完全二叉树

<!-- more -->

## 支持函数

```cpp
// 堆内元素个数
int size()
{
    ...
}

// 获取父节点的值
int top()
{
    ...
}

// 写入一个值
void push(int val)
{
    ...
}

// 将父节点值弹出
void pop(int val)
{
    ...
}

```

## 实现

话不多说，我们直接开撕。

* heap.h
```cpp
#ifndef UNTITLED_HEAP_H
#define UNTITLED_HEAP_H

#include <vector>
#include <iostream>

using namespace std;

template<class T>
class heap {
public:
    heap()
    {
        _heap.push_back(0);
    }

    int size()
    {
        return _heap.size() - 1;
    }

    bool empty()
    {
        return size() == 0;
    }

    void push(T val)
    {
        _heap.push_back(val);
        auto index = size();

        while (index / 2 > 0 && _heap[index] > _heap[index / 2])
        {
            swap(_heap[index], _heap[index / 2]);
            index /= 2;
        }
    }

    void pop()
    {
        if (empty())
            return;

        int heap_size = size();
        swap(_heap[1], _heap[heap_size]);
        _heap.pop_back();
        heap_size--;

        int index = 1;
        while (true)
        {
            int maxI = index;

            if (2 * index <= heap_size && _heap[index] < _heap[2 * index])
                maxI = 2 * index;
            else if (2 * index + 1 < heap_size && _heap[maxI] < _heap[2 * index])
                maxI = 2 * index + 1;

            if (maxI == index)
                break;
            swap(_heap[index], _heap[maxI]);
            index = maxI;
        }
    }
    void print()
    {
        for (int i = 1; i < _heap.size(); i++)
        {
            cout << _heap[i] << " ";
        }
        cout << endl;
    }

private:
    vector<T> _heap;
};

#endif //UNTITLED_HEAP_H

```

* main.cpp
```cpp
#include "heap.h"

int main() {
    heap<int> bigheap;
    bigheap.push(4);
    bigheap.push(3);
    bigheap.push(2);
    bigheap.push(1);
    bigheap.push(5);
    bigheap.print();
    bigheap.pop();
    bigheap.print();
    bigheap.pop();
    bigheap.print();

    return 0;
}

```

* CMakeLists.txt
```
cmake_minimum_required(VERSION 3.19)
project(untitled)

set(CMAKE_CXX_STANDARD 14)

add_executable(untitled main.cpp)

```

## 4. 运行

* 编译

```shell
$ cmake -DCMAKE_BUILD_TYPE=Debug -G "CodeBlocks - Unix Makefiles" ./
$ cmake --build ./ --target untitled
```

* 运行

```shell
$ ./untitled
5 4 2 1 3 
4 3 2 1 
3 1 2 

```

## 5. 总结

我们这里实现了一个大顶堆。聪明的你一定知道了小顶堆应该如何实现啦。

那么这里提出一个小问题，快排和堆都可以作为排序算法，

那么为什么我们在实际工程项目中快排的应用更加广泛呢？（或许可以从运行结果中发现端倪哦）

