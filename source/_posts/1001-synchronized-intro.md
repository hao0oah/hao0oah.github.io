---
title: synchronized入门和实现原理
date: 2024-07-06 19:19:29
tags:
    - Java
    - synchronized
    - 多线程
    - 并发编程
categories: Java
---

### synchronized 简介

`synchronized` 是 Java 中的一种同步机制，用于防止多个线程同时访问共享资源。它可以用于方法和代码块，确保同一时间只有一个线程可以执行被 `synchronized` 保护的代码，同时还可以保证共享变量的内存可见性。

Java中每一个对象都可以作为锁，这是`synchronized`实现同步的基础。

<!-- more -->

### synchronized 的特性

**原子性**：`synchronized` 保证了被保护的代码块在一个线程执行期间不会被其他线程中断。

**可见性**：`synchronized` 确保在锁释放前对共享变量的修改对其他线程可见。

**有序性**：`synchronized`保证同一时刻只能有一个线程进行操作，也就保证了有序性，有效解决指令重排对程序的影响。

**重入性**：同一个线程可以多次获取同一把锁而不会被阻塞。


### synchronized的使用

#### 同步实例方法

```java
public class ThreadTest implements Runnable{
    // 共享资源(临界资源)
    static int i=0;

    // 如果没有synchronized关键字，输出小于20000
    public synchronized void increase(){
        i++;
    }
    public void run() {
        for(int j=0;j<10000;j++){
            increase();
        }
    }
    public static void main(String[] args) throws InterruptedException {
        ThreadTest t = new ThreadTest();
        Thread t1 = new ThreadTest(t);
        Thread t2 = new ThreadTest(t);
        t1.start();
        t2.start();
        t1.join(); // 主线程等待t1执行完毕
        t2.join(); // 主线程等待t2执行完毕
        System.out.println(i); // 输出结果:20000
    }
}
```

#### 修饰静态方法

```java
public class ThreadTest {
    // 共享资源(临界资源)
    static int i = 0;

    // 如果没有synchronized关键字，输出小于20000
    public static synchronized void increase() {
        i++;
    }

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(new Runnable() {
            public void run() {
                for (int j = 0; j < 10000; j++) {
                    increase();
                }
            }
        });
        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                for (int j = 0; j < 10000; j++) {
                    increase();
                }
            }
        });
        t1.start();
        t2.start();
        t1.join(); // 主线程等待t1执行完毕
        t2.join(); // 主线程等待t2执行完毕
        System.out.println(i); // 输出结果:20000
    }
}
```

#### 修饰代码块

```java
public class ThreadTest implements Runnable{
    // 共享资源(临界资源)
    static int i=0;

    @Override
    public void run() {
        for(int j=0;j<10000;j++){
            // 获得了ThreadTest的类锁
            synchronized (ThreadTest.class){
            	i++;
            }
        }
    }
    public static void main(String[] args) throws InterruptedException {
        ThreadTest t = new ThreadTest();
        Thread t1 = new Thread(t);
        Thread t2 = new Thread(t);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(i); // 输出结果:20000
    }
}
```

1. `synchronized`修饰的是实例方法时，获得该实例对象的内置锁。
2. `synchronized`修饰的时静态方法时，获得类锁，即当前类的Class对象的内置锁。
3. `synchronized`修饰的是代码块时，根据括号中的对象或者类，获得相应的对象内置锁或者类锁。
4. 每个类都有一个类锁，类的每个对象都有一个内置锁，它们是互不干扰的，也就是说一个线程可以同时获得类锁和该类实例化对象的内置锁。

### synchronized的实现原理

`synchronized` 的底层实现依赖于` JVM` 中的对象头和 `Monitor`对象。

- **对象头**：每个 Java 对象都有一个对象头，包含了锁标志位和指向 `Monitor` 对象的指针。

- **Monitor对象**：`Monitor` 是 JVM 内部的一个同步工具，用于管理线程的同步。

#### 锁的状态

- **无锁状态**：对象未被锁定。

- **偏向锁**：锁偏向于第一个获取它的线程，降低了无竞争情况下的加锁和解锁成本。

- **轻量级锁**：多个线程竞争锁时，锁会膨胀为轻量级锁。

- **重量级锁**：竞争激烈时，锁会膨胀为重量级锁，使用操作系统的 `Mutex` 进行管理。

锁的强度会随着多线程竞争的激烈而逐渐升级。

#### synchronized的字节码实现

- 通过下面的测试代码来分析：
  
  ```java
  public class SynchronizedDemo {
      private int i=1;
      public void test1(){
          synchronized (this){
              i++;
          }
      }
      public synchronized void test2(){
          i++;
      }
  }
  ```

- 执行`javap -verbose SynchronizedDemo.class `反编译

其中`test1()`方法的字节码如下：

  ```java
    public void test1();
      descriptor: ()V
      flags: ACC_PUBLIC
      Code:
        stack=3, locals=3, args_size=1
           0: aload_0
           1: dup
           2: astore_1
           3: monitorenter                      // 申请获得对象的内置锁
           4: aload_0
           5: dup
           6: getfield      #2                  // Field i:I
           9: iconst_1
          10: iadd
          11: putfield      #2                  // Field i:I
          14: aload_1
          15: monitorexit                       // 释放对象内置锁
          16: goto          24
          19: astore_2
          20: aload_1
          21: monitorexit                       // 释放对象内置锁（同步块内发生异常时）
          22: aload_2
          23: athrow
          24: return
        Exception table:
           from    to  target type
               4    16    19   any
              19    22    19   any
  // 异常表确保在 4 到 16 字节码位置之间发生异常时，跳转到 19 位置进行异常处理。这段代码在发生异常时也会执行 monitorexit，确保锁被释放，避免了死锁和锁泄露问题。
  ```

`test1()`使用了`synchronized`代码块，字节码中通过`monitorenter`指令获取锁，通过`monitorexit`指令释放锁。注意到其中只有1个`monitorenter`但是有2个`monitorexit`，因为`synchronized`是JVM层面的锁，如果发生异常了JVM会自动帮我们释放锁，而Lock异常则需要我们手动捕获并释放。

关于这两条指令的作用，我们直接参考JVM规范中的描述：

- `monitorenter`

```
Each object is associated with a monitor. A monitor is locked if and only if it has an owner. The thread that executes monitorenter attempts to gain ownership of the monitor associated with objectref, as follows:
• If the entry count of the monitor associated with objectref is zero, the thread enters the monitor and sets its entry count to one. The thread is then the owner of the monitor.
• If the thread already owns the monitor associated with objectref, it reenters the monitor, incrementing its entry count.
• If another thread already owns the monitor associated with objectref, the thread blocks until the monitor's entry count is zero, then tries again to gain ownership.

monitorenter翻译如下：
每个对象都有一个监视器锁（monitor）。当monitor被占用时就会处于锁定状态，线程执行monitorenter指令时尝试获取monitor的所有权，过程如下：
• 如果monitor的进入数为0，则该线程进入monitor，然后将进入数设置为1，该线程即为monitor的所有者。
• 如果线程已经占有该moni tor，只是重新进入，则进入monitor的进入数加1。
• 如果其他线程已经占用了monitor，则该线程进入阻塞状态，直到monitor的进入数为0，再重新尝试获取monitor的所有权。
```

- `monitorexit`

```
The thread that executes monitorexit must be the owner of the monitor associated with the instance referenced by objectref.
The thread decrements the entry count of the monitor associated with objectref. If as a result the value of the entry count is zero, the thread exits the monitor and is no longer its owner. Other threads that are blocking to enter the monitor are allowed to attempt to do so.

执行monitorexit的线程必须是objectref所对应的monitor的所有者。指令执行时，monitor的进入数减1，如果减1后进入数为0，那线程退出monitor，不再是这个monitor的所有者。其他被这个monitor阻塞的线程可以尝试去获取这个 monitor的所有权。 
```

通过这两段描述，我们应该能很清楚的看出`synchronized`的实现原理，`synchronized`的语义底层是通过一个`monitor`的对象来完成，其实`wait()`/`notify()`等方法也依赖于`monitor`对象，这就是为什么只有在同步块或者方法中才能调用`wait()`/`notify()`等方法，否则会抛出`java.lang.IllegalMonitorStateException`异常的原因。

`test2()`方法的字节码如下:

```java
public synchronized void test2();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: dup
         2: getfield      #2                  // Field i:I
         5: iconst_1
         6: iadd
         7: putfield      #2                  // Field i:I
        10: return
```

`test2()`使用了`synchronized`方法，虽然在字节码中并没有看到`monitorenter`和`monitorexit`指令，但在字节码常量池中多了`ACC_SYNCHRONIZED`标志符，JVM就是根据该标示符来实现方法同步的。当方法调用时，调用指令会检查该标志符是否被设置，如果设置了，执行线程将先获取`monitor`，获取成功之后才能执行方法体，方法执行完后再释放`monitor`。在方法执行期间，其他任何线程都无法再获得同一个`monitor`对象。 其实本质上没有区别，只是方法的同步是一种隐式的方式来实现。

### 总结

`synchronized`是Java并发编程中最常用的用于保证线程安全的方式，使用起来非常简单和方便。但是如果能够深入了解其原理，对监视器锁等底层知识有所了解，一方面可以帮助我们正确的使用`synchronized`关键字，另一方面也能够帮助我们更好的理解并发编程机制，有助于我们在不同的情况下选择更优的并发策略来完成任务。对平时遇到的各种并发问题，也能够从容的应对。

### 参考

https://mp.weixin.qq.com/s/2yxexZUr5MWdMZ02GCSwdA

https://www.cnblogs.com/paddix/p/5367116.html