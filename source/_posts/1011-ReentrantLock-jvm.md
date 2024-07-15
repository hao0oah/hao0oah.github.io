---
title: 从jvm源码看ReentrantLock实现原理
date: 2024-07-12 19:19:29
tags:
    - Java
    - AQS
    - ReentrantLock
    - 多线程
    - 并发编程
categories: Java
---


JAVA里的锁有两大类，一类是`synchronized`锁，一类是`concurrent`包里的锁（`JUC锁`）。其中`synchronized`锁是JAVA语言层面提供的能力，本文主要讨论`JUC`里的`ReentrantLock`锁。

<!-- more -->


### JDK层

#### AbstractQueuedSynchronizer(AQS)
`ReentrantLock`的`lock()`、`unlock()`等API其实依赖于内部的`Synchronizer`（**注意不是**`synchronized`）来实现。`Synchronizer`又分为`FairSync`和`NonfairSync`，顾名思义即公平和非公平。

当调用`ReentrantLock`的`lock()`方法时，其实就只是简单地转交给`Synchronizer`的`lock()`方法。

- `ReentrantLock`的主要方法和结构如下：

```java
public class ReentrantLock implements Lock, java.io.Serializable {
    
    /** Synchronizer providing all implementation mechanics */
    private final Sync sync;
    
    /**
     * Base of synchronization control for this lock. Subclassed
     * into fair and nonfair versions below. Uses AQS state to
     * represent the number of holds on the lock.
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {...}
    
    /**
     * Sync object for non-fair locks
     */
    static final class NonfairSync extends Sync {
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
    
    /**
     * Sync object for fair locks
     */
    static final class FairSync extends Sync {
        final void lock() {
            acquire(1);
        }
    }
    
    // ReentrantLock默认构造方法(非公平锁)，相当于ReentrantLock(false)
    public ReentrantLock() {
        sync = new NonfairSync();
    }
    // ReentrantLock构造方法(true-公平;false-非公平)
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
    // 加锁方法，调用FairSync/NonfairSync的lock()->AQS.acquire()->FairSync/NonfairSync的tryAcquire()
    public void lock() {
        sync.lock();
    }
    // 释放锁，实际调用sync的父类AQS.release()->Sync.tryRelease()
    public void unlock() {
        sync.release(1);
    }
... ...
}
```

我们看到`Sync` 继承自`AbstractQueueSynchronizer`(`AQS`)，`AQS`是`JUC`包的基石，`AQS`本身并不实现任何同步接口（比如`lock`、`unlock`、`countDown`等等），但是它定义了一个并发资源控制逻辑的框架（运用了`template method`设计模式），它定义了`acquire()`和`release()`方法用于独占地（`exclusive`）获取和释放资源，以及`acquireShared()`和`releaseShared()`方法用于共享地获取和释放资源。比如`acquire()`/`release()`用于实现`ReentrantLock`，而`acquireShared`/`releaseShared`用于实现`CountDownLacth`、`Semaphore`。

- `AQS.acquire()`定义如下：

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

整体逻辑是，先进行一次`tryAcquire()`，如果成功了，调用者继续执行自己后面的代码；如果失败，则执行`addWaiter()`和`acquireQueued()()`。其中`tryAcquire()`需要子类根据自己的同步需求进行实现，而`acquireQueued()` 和`addWaiter() `已经由AQS实现。`addWaiter()`的作用是把当前线程加入到`AQS`内部同步队列的尾部，而`acquireQueued()`的作用是当`tryAcquire()`失败的时候阻塞当前线程。

- `addWaiter()`的代码如下：

```java
private Node addWaiter(Node mode) {
    // 创建节点,设置关联线程和模式(独占或共享)
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    // 如果尾节点不为空，说明同步队列已经初始化过
    if (pred != null) {
        // 新节点的前驱节点设置为尾节点
        node.prev = pred;
        // 设置新节点为尾节点
        if (compareAndSetTail(pred, node)) {
            // 老的尾节点的后继节点设置为新的尾节点，所以同步队列是一个双向列表。
            pred.next = node;
            return node;
        }
    }
    // 如果尾节点为空，说明队列还未初始化，需要初始化head节点并加入新节点
    enq(node);
    return node;
}
```

- `enq(node)`的代码如下：

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            // 如果tail为空，则新建一个head空节点，并且tail和head都指向这个head节点
            // 队列头节点称作“哨兵节点”或者“哑节点”，它不与任何线程关联
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            // 第二次循环进入这个分支，将node节点加入队列尾部
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

- 同步队列的结构如下：

![AQS同步队列](https://s2.loli.net/2024/07/12/kE5GONbBiwRmnu9.jpg)

- `acquireQueued()`的代码如下：

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            // 获取当前node的前驱node
            final Node p = node.predecessor();
            // 如果前驱node是head node，说明自己是第一个排队的线程，则尝试获锁
            if (p == head && tryAcquire(arg)) {
                // 把获锁成功的当前节点变成head node
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

判断自己是不是同步队列中的第一个排队的节点，如果是则尝试进行加锁，如果成功，则把自己变成`head node`，过程如下所示：

![](https://s2.loli.net/2024/07/12/oOMYHA1yCQ6GTSj.jpg)

如果自己不是第一个排队的节点或者`tryAcquire()`失败，则调用``shouldParkAfterFailedAcquire()`，其主要逻辑是使用`CAS`将节点状态由` INITIAL` 设置成 `SIGNAL`，表示当前线程阻塞等待`SIGNAL`唤醒。如果设置失败，会在 `acquireQueued()`方法中的死循环中继续重试，直至设置成功。然后调用`parkAndCheckInterrupt()` 方法，把当前线程阻塞挂起，等待唤醒。`parkAndCheckInterrupt()`的实现需要借助下层的能力，这是本文的重点，在下文中逐层阐述。
