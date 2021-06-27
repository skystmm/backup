---
title: AQS源码分析
date: 2018-05-15 22:45:58
tags: [java,J.U.C]
---

# AQS源码分析

## AbstractQueuedSynchronizer类结构分析

![AbstractQueuedSynchronizer](http://image.stxl117.top/AQS.png)

其中AbstractOwnaleSynchronizer提供了当前资源拥有者相关的操作，AbatractQueuedSynchronizer（下文中简称为AQS）这个抽象类中主要提供了对互斥锁和共享锁相关操作提供了基础功能的实现，以及提供了对于不同场景下的加锁和释放锁的方法定义。

而AQS这个抽象类中主要为我们提供了对于CLH队列的一系列操作，包括无法获得请求资源时的如队列操作和资源释放时怎么通知等待队列中的节点获取资源的操作。这样使得我们在需要自己实现锁功能时，只要需要专注于具体的加锁和释放锁操作。下面来看一下AQS中核心的数据结构CLH队列。

<!-- more -->

## CLH队列

上面基础数据结构为Node，Node为一个双向链表结构。CLH中节点数据结实现如下：
```java

static final class Node {

    static final Node SHARED = new Node();
    static final Node EXCLUSIVE = null;
    static final int CANCELLED = 1;
    static final int SIGNAL = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;

    volatile int waitStatus;
    volatile Node prev;
    volatile Node next;
    volatile Thread thread;

    Node nextWaiter;

    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    Node() {    // Used to establish initial head or SHARED marker
    }

    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}

```
可以看出CLH队列是一个双线链表结构，每个节点包含线程信息和状态信息。


## 独占模式下的CLH操作


### 1.加锁

```java

public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
}
```
通过AQS的源码可以看到获取独享锁时，首先调用tryAcquire方法进行锁获取，而AQS源码中我们发现tryAcuire方法为一个抽象方法，所以获取锁资源是根据具体的业务要求，有具体类进行实现的。而获取锁失败后，首先执行addWaiter方法，addWaiter方法是将当前请求线程信息构造成CLH队列中的节点，然后再将节点插入到队列的队尾，同时返回创建的这个node节点，然后执行acquireQueued方法，下面看下acquireQueued的实现，

```java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
方法中，可以看到如果新加的节点是第一个节点，将再尝试获取获取锁一次，如果获取成功，进行将其从队列中移除，并返回。如果不是第一个节点，则设置节点状态，并挂起该线程。


大致加锁流程为：

### 2.释放锁

```java
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
和获取资源一样，释放资源的时候，还是先调用tryRelease方法进行资源释放，同样的tryRealse方法也是抽象方法，需要在具体实现类中进行具体实现的。当资源释放后，且CLH等待队列不为空时，会进行队列中节点唤醒的操作，具体实现如下：

```java
private void unparkSuccessor(Node node) {

        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```
可以看到唤醒时有两种情况，正常情况下是唤醒头结点后的第一个节点，当头结点下一个节点为空时后者该节点状态不为SIGNAL时，将从尾节点开始开始向前遍历找到排在最前面的一个为SIGNAL的节点，唤醒对应的线程进行锁资源的获取。

大体流程如下图所示：



> 具体实现tryAcquire和tryRelease可以参考Juc包中的ReentrantLock类

## 共享模式下的CLH操作


### 1.加锁

```java

 public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```
同样的我们需要在具体的场景下实现tryAcquireShared方法，来实现加锁的过程。而AQS则我们实现了，未获取到所情况时，对于获取锁失败的线程的等待操作。

```java
private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
可以看到共享锁的未获取到资源情况的操作与独占锁类似，都是将该线程挂起，并创建对应的Node节点插入到CLH等待队列中。


### 2.释放锁

```java

public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }

private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

可以看到共享锁的未获释放资源情况的操作与独占锁类似。



> 具体实现tryAcquire和tryRelease可以参考Juc包中的ReentrantReadWriteLock类中读锁的实现


总的来说AQS用模板方法的方式，定义了通过Cas获取和释放锁的流程，以及对于等待节点的操作。而我们在使用已AQS为框架自实现锁时，只需要专注于获取锁和释放锁的具体细节实现，即可完成自定义锁的功能。
