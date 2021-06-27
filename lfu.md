---
title: 从lintcode LFU引发的思考
date: 2020-02-18 21:56:34
tags: [java,源码,刷题]
categories: java
---

## 题目24. LFU Cache

> LFU是一个著名的缓存算法
>对于容量为k的缓存，如果缓存已满，并且需要逐出其中的密钥，则最少使用的密钥将被踢出。
>实现LFU中的set 和 get操作

题目的内容比较直观就是设计一个LFU缓存淘汰算法的实现，下面我们先来看下LFU的基本流程


### LFU
LFU算法中会记录一定时间内每个Key的被访问次数，淘汰时会淘汰最近一段时间内被访问次数最少的key,当多个keys计数值为最小值时，按照LRU进行淘汰,如下图所示：
![LFU示意](http://image.stxl117.top/lfu.png)
<!-- more -->
## 其他常见缓存淘汰策略

### FIFO
FIFO是最简单的先进先出，使用Queue即可实现。
### LRU
LRU是根据最近最少使用原则进行缓存淘汰的算法。顾名思义当需要进行页置换或者缓存淘汰时将最近最少命中的页或缓存剔除的算法，如下图：
![LRU流程](http://image.stxl117.top/lru.png)

## 解题思路

### HashMap + 小根堆
时间复杂度 get : O(1)  set O(logk)
为了查询速度将缓存已key-value的形式保存在hashMap中，同时使用数组实现的小根堆来记录数据的访问频次情况，以便于淘汰策略进行淘汰操作。

>[LFU解题代码](https://github.com/skystmm/AlgorithmExercise/tree/master/src/main/java/com/skystmm/lintcode/cache)

## java中优先级队列PriorityQueue解析

`PriorityQueue`是一个无界的按照给定的比较方式进行小根堆排序的队列实现。

![PriorityQueue类关系](http://image.stxl117.top/PQ.jpg)

从类图上我们可以看到，`PriorityQueue`的基础数据结构是`Object[]`，而通过`comparator`属性提供个性化的排序要求。由于上文中提到过`PriorityQueue`中是通过比较实现的小根堆。

>之后会把PriorityQueue简称为PQ

### 1. 构造方法
```java
    public PriorityQueue() {
        this(DEFAULT_INITIAL_CAPACITY, null);
    }

    public PriorityQueue(int initialCapacity) {
        this(initialCapacity, null);
    }

    public PriorityQueue(Comparator<? super E> comparator) {
        this(DEFAULT_INITIAL_CAPACITY, comparator);
    }

    public PriorityQueue(int initialCapacity,
                         Comparator<? super E> comparator) {
        // Note: This restriction of at least one is not actually needed,
        // but continues for 1.5 compatibility
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.queue = new Object[initialCapacity];
        this.comparator = comparator;
    }


    @SuppressWarnings("unchecked")
    public PriorityQueue(Collection<? extends E> c) {
        if (c instanceof SortedSet<?>) {
            SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
            this.comparator = (Comparator<? super E>) ss.comparator();
            initElementsFromCollection(ss);
        }
        else if (c instanceof PriorityQueue<?>) {
            PriorityQueue<? extends E> pq = (PriorityQueue<? extends E>) c;
            this.comparator = (Comparator<? super E>) pq.comparator();
            initFromPriorityQueue(pq);
        }
        else {
            this.comparator = null;
            initFromCollection(c);
        }
    }


    @SuppressWarnings("unchecked")
    public PriorityQueue(PriorityQueue<? extends E> c) {
        this.comparator = (Comparator<? super E>) c.comparator();
        initFromPriorityQueue(c);
    }

    @SuppressWarnings("unchecked")
    public PriorityQueue(SortedSet<? extends E> c) {
        this.comparator = (Comparator<? super E>) c.comparator();
        initElementsFromCollection(c);
    }
```
PQ类提供了上面代码中的构造函数，从构造方法中，可以发现PQ中有两个比较重要的`capacity`和`comparator `属性（上文中已经提到过作用了）。capactity用来表示队列的容量，可以上面的代码中看到PQ的默认队列容量为DEFAULT_INITIAL_CAPACITY(11)，同时还提供了动态扩容的实现；对于通过已有其他Collection的实现类构造PQ队列时，基本可以分为两个步骤进行处理,首先将数据赋值给PQ的Object[]数组中，然后通过heapify方法对PQ Object[]数组的每个元素记性siftDown操作，来达到数据按照我们的约定的方式构建成相应的小根堆。

下面我们就来看看siftDown方法是怎么对已有数据进行堆排序的数据调整的,话不多说，代码先行。
```java
   private void siftDown(int k, E x) {
        if (comparator != null)
            siftDownUsingComparator(k, x);
        else
            siftDownComparable(k, x);
    }

    @SuppressWarnings("unchecked")
    private void siftDownComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>)x;
        int half = size >>> 1;        // loop while a non-leaf
        while (k < half) {
            int child = (k << 1) + 1; // assume left child is least
            Object c = queue[child];
            int right = child + 1;
            if (right < size &&
                ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
                c = queue[child = right];
            if (key.compareTo((E) c) <= 0)
                break;
            queue[k] = c;
            k = child;
        }
        queue[k] = key;
    }
```
`siftDown`方法接收两个参数，i为开始比较的位置，x为比较元素。上面没有把`siftDownUsingComparator`方法代码放上去，是因为两者的逻辑不同处仅在于比较上的不同，整体逻辑是一样的。由于PQ中的优先级排序方式采用的是小根堆的排序方式，可以看到`siftDownUsingComparator`主要操作是队列中的当前节点与其叶子节点的比较和交换操作。总体来说`siftDown`方法是对已有队列进行在排序的操作，在`removeAt`和`poll`被调用。

### 2. 插入元素
Queue提供了`add`方法进行队列的入队操作，下面看看PQ中add的实现
```java
   public boolean add(E e) {
        return offer(e);
    }
    public boolean offer(E e) {
        if (e == null)
            throw new NullPointerException();
        modCount++;
        int i = size;
        if (i >= queue.length)
            grow(i + 1);
        size = i + 1;
        if (i == 0)
            queue[0] = e;
        else
            siftUp(i, e);
        return true;
    }
   public boolean add(E e) {
         return offer(e);
    }
    public boolean offer(E e) {
        if (e == null)
            throw new NullPointerException();
        modCount++;
        int i = size;
        if (i >= queue.length)
            grow(i + 1);
        size = i + 1;
        if (i == 0)
            queue[0] = e;
        else
            siftUp(i, e);
        return true;
    }
```
代码中可以看到，执行add操作流程可以看到，PQ中不接受null值。在入队前会先对队列容量进行判定，如果容量不足，将会触发扩容(`grow`方法)。扩容的代码这里就不放出来了。从插入第二个元素开始就会调用`siftUp`方法,来进行元素的堆排序操作（如下代码）。
```java
private void siftUpComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>) x;
        while (k > 0) {
            int parent = (k - 1) >>> 1;
            Object e = queue[parent];
            if (key.compareTo((E) e) >= 0)
                break;
            queue[k] = e;
            k = parent;
        }
        queue[k] = key;
    }
```
>扩容的操作中，当队列当前容量小于64时，扩容为容量的两倍，大于64时，扩容为当前容量的1.5倍，当容量超过`MAX_ARRAY_SIZE`的时候，会触发`hugeCapacity`进行扩容，如果需要的容量超过`MAX_INTEGER`会抛出oom异常，当所需容量大于`MAX_ARRAY_SIZE`时扩容为`MAX_INTEGER`。

### PQ的总结
总的来说PQ是一个基于小根堆排序方式实现的优先级队列， 队列中的第一个元素为当前排序方式中最小的元素，遍历结果并不是完全有序的（堆排序特点）。PQ中操作的时间复杂度`offer`,`add`,`poll`,`remove`为o(logn);`remove(item)`,`contain`操作为 o(n),`peek`,`获取某个元素`,`size`为o(1)。根据以上特点我们可以在按照优先级对任务进行分类执行时，使用PQ来进行任务的接收；当然LFU也可以使用PQ来实现。
