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

`synchronized` 是 Java 中的一种同步机制，用于防止多个线程同时访问共享资源。它可以用于方法和代码块，确保同一时间只有一个线程可以执行被 `synchronized` 保护的代码。

### synchronized 的特性

**原子性**：`synchronized` 保证了被保护的代码块在一个线程执行期间不会被其他线程中断。

**可见性**：`synchronized` 确保在锁释放前对共享变量的修改对其他线程可见。

**重入性**：同一个线程可以多次获取同一把锁而不会被阻塞。

<!-- more -->


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
2. `synchronized`修饰的时静态方法时，获得类锁，即该Class对象的内置锁。
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
  
  通过`javap -verbose`反编译，结果如下：
  
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

`test1()`使用了`synchronized`代码块，字节码中通过`monitorenter`获取锁，通过`monitorexit`释放锁。

`test2()`使用了`synchronized`方法，字节码中通过`ACC_SYNCHRONIZED`标志，在JVM内部实现。

`synchronized` 通过进入和退出 `Monitor` 对象来实现同步。进入 `Monitor` 的过程包括检查锁的状态、尝试获取锁、阻塞线程等步骤。