---
title: Java并发编程(三)：ReentrantLock
date: 2022-02-05 17:35:40
tags:
---

ReentrantLock是可以用来代替synchronized的。ReentrantLock比synchronized更加灵活，功能上面更加丰富，性能方面自synchronized优化后两者性能没有什么太大差别。

说一下两者的区别首先ReetrantLock是基于JDK实现层面的，而synchronized是基于JVM层面实现的。ReentrantLock可以进行tryLock尝试锁定，支持公平锁的实现。
```java
Lock lock = new ReentrantLock(); 
lock.lock();
try {
    //业务逻辑  
} finally {
  lock.unlock();
}
```
 一个ReentrantLock的简单实现，要注意的是，必须要手动释放锁，不然很容易产生死锁。使用sync锁定的话遇到异常，jvm会自动释放锁，但是reentrantLock必须手动释放锁，因此经常在finally中进行锁的释放。

```java
Lock lock=new ReentrantLock();
    boolean locked=false;
    try{
        try {
            locked=lock.tryLock(5,TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }finally {
        if(locked){
            lock.unlock();
        }
    }
```
ReentrantLock可以进行tryLock尝试锁定，tryLock的方法就是进行尝试锁定，如果能在指定时间内得到锁，就返回true，反之返回false。指定时间内无法锁定时，线程可以决定是否继续等待。

```java
private static ReentrantLock lock=new ReentrantLock(true); //参数为true表示公平锁
```
ReentrantLock可以指定为公平锁，ReentrantLock和sync默认是非公平锁，**非公平锁**：线程加锁时**直接尝试获取锁**，获取不到就自动到队尾等待。而**公平锁**：加锁前先查看是否有排队等待的线程，有的话优先处理排在前面的线程，**先来先得**。
