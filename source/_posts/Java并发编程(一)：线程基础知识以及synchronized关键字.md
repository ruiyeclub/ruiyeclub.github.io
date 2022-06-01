---
title: Java并发编程(一)：线程基础知识以及synchronized关键字
date: 2022-02-03 17:35:40
tags:
---

1.线程与多线程的概念：在一个程序中，能够独立运行的程序片段叫作“*线程*”（Thread）。*多线程*（multithreading）是指从软件或者硬件上实现多个线程并发执行的技术。

2.多线程的意义：多线程可以在时间片里被cpu快速切换，资源能更好被调用、程序设计在某些情况下更简单、程序响应更快、运行更加流畅。

2.如何启动一个线程：继承Thread类、实现Runnable接口、实现Callable接口

3.为什么要保证线程的同步？：java允许多线程并发控制，当多个线程同时操作一个可共享的资源变量时（如数据的增删改查），将会导致数据不准确，相互之间产生冲突，因此加入同步锁以避免在该线程没有完成操作之前，被其他线程的调用，从而保证了该变量的唯一性和准确性。

4.基本的线程同步：使用*synchronized*关键字、特殊域变量volatile、wait和notify方法等。

```java
public class T {
    private int count=10;
    private Object o=new Object();

    public void m(){
        synchronized (o){
            //任何线程要执行下面的代码，必须先拿到o的锁
            count--;
            System.out.println(Thread.currentThread().getName()+"count="+count);
        }
    }
}

```
假设这段代码中有多个线程，当第一个线程执行到m方法来的时候，sync锁住了堆内存中的o对象，这个时候第二个线程是进不来的，它必须等第一个线程执行完，锁释放掉才可以接着执行。这里有个锁的概念叫做互斥锁，sync是一种*互斥锁*。
```java
public class T1 {
    private static int count =10;

    public synchronized static void m(){
        //这里等同于synchronized(T3.class)
        count--;
        System.out.println(Thread.currentThread().getName()+"count="+count);
    }

    public static void mm(){
        synchronized (T1.class){
            //思考：这里写成synchronized(this)是否可以？
            count--;
        }
    }
}
```
思考这里，注意sync修饰的是一个静态方法和静态的属性，静态修饰的方法和属性是不需要new出对象来就可以访问的，所以这里没有new出对象，sync锁定的是T3.class对象。

```java
public class T2 implements Runnable{
    private int count =10;

    @Override
    public /*synchronized*/ void run(){
        count--;
        System.out.println(Thread.currentThread().getName()+"count="+count);
    }

    public static void main(String[] args) {
        T2 t=new T2();
        for (int i=0;i<5;i++){
            new Thread(t,"THREAD"+i).start();
        }
    }
}
```
上面代码中开启了五个线程，执行run方法对count进行减一的操作，这里会出现线程抢占资源的问题。当第一个线程在执行run方法时，减减的过程中，第二个线程也进入了方法，同样也在执行减减，可能会出现第二个线程减完的时候，第一个线程才输出count，这时候就出现了线程重入。处理方法可以加上synchronized关键字，只有等第一个线程执行完毕，第二个线程才可以进入，即同一时刻只能有一个线程来对它进行操作，也就是*原子性*。

```java
public class Account {

    /**
     * 账号持有人和账号余额
     */
    String name;
    double balance;

    /**
     * 设置账号余额，线程睡眠目的是先让它读数据
     * @param name
     * @param balance
     */
    public synchronized void set(String name,double balance){
        this.name=name;

        try {
            Thread.sleep(2000);
        }catch (InterruptedException e){
            e.printStackTrace();
        }
        this.balance=balance;
    }

    public double getBalance(String name){
        return this.balance;
    }

    public static void main(String[] args) {
        Account a=new Account();
        new Thread(()->a.set("zhangsan",100.0)).start();

        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(a.getBalance("zhangsan"));

        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(a.getBalance("zhangsan"));
    }
}
```
这是一个小demo，代码中只对set方法加了锁，没有对get方法加锁，这个时候会出现*脏读*现象。解决方法是读和写的方法都加锁。

```java
public class T3 {

    synchronized void m1(){
        System.out.println("m1 start");
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        m2();
    }

    synchronized void m2() {
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("m2");
    }
}
```
这个案例中，m1和m2方法都已经加上了锁，当执行m1的时候再去执行m2，这样是可以的。一个同步方法可以调用另外一个同步方法，也就是说sync获得的锁是*可重入锁*。还有个概念是*死锁*，死锁是指多个线程抢占资源而造成的一种互相等待。举个例子，一个箱子需要两把钥匙才可以打开，钥匙分别在两个人手中，这两个人互相抢占另外一个人的钥匙，导致箱子打不开。