---
title: Java并发编程(二)：volatile关键字
date: 2022-02-04 17:35:40
tags:
---

volatile是Java虚拟机提供的轻量级的同步机制。volatile关键字有如下两个作用，一句话概括就是**内存可见性**和禁止重排序。
1）保证被volatile修饰的共享变量对所有线程总是可见的，也就是**当一个线程修改了一个被volatile修饰共享变量的值，新值总是可以被其他线程立即得知。**
2）禁止指令重排序优化。在执行程序时为了提高性能，编译器和处理器通常会重新安排指令的执行顺序。
```java
public class T {

    /*volatile*/ boolean running=true;

    void m(){
        System.out.println("m start");
        while (running){}
        System.out.println("m end");
    }

    public static void main(String[] args) {
        T t=new T();

        new Thread(()->t.m(),"t1").start();

        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        t.running=false;
    }
}
```
这段代码中没加volatile的话，循环的后面的输出语句"m end"不会输出，但是main方法里面确实是已经将running的值改成了false。这里有关java的**内存模型**，简单来说就是程序运行时，running的值已经被加载到了缓存里面，而改的是t对象的running值在堆内存中。加了volatile就不一样了，它改完数值之后，会通知其他线程，让其他线程重新读一下堆内存中的值，也就是内存可见性。

```java
public class T1 {

    volatile int count=0;

    /*synchronized*/ void m(){
        for (int i=0;i<10000;i++){
            count++;
        }
    }

    public static void main(String[] args) {

        T1 t=new T1();
        List<Thread> threads=new ArrayList<Thread>();

        for(int i=0;i<10;i++){
            threads.add(new Thread(()->t.m(),"thread-"+i));
        }
        threads.forEach((o)->o.start());
        threads.forEach((o)->{
            try {
                o.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        System.out.println(t.count);
    }
}
```
这段代码中，十个线程执行count++加到1w循环，理想状态当然是加到10w，但是volatile只能保持线程的可见性，不能保持原子性。给count变量加上了volatile，最后输出的count值也达不到10w。这就是volatile与synchronized的区别，**volatile只能保持可见性，而synchronized可以保持可见性和原子性。**但是volatile更加轻量级，能用volatile的情况下，尽量别用synchronized。