---
title: ReentrantLock入门和实践
date: 2024-07-11 19:19:29
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

