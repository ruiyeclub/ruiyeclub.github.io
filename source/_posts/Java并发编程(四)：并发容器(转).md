---
title: Java并发编程(四)：并发容器(转)
date: 2022-02-06 17:35:40
tags:
---

解决并发情况下的容器线程安全问题的。给多线程环境准备一个线程安全的容器对象。
线程安全的容器对象: Vector, Hashtable。线程安全容器对象，都是使用synchronized 方法实现的。concurrent包中的同步容器，大多数是使用系统底层技术实现的线程安全。类似native。Java8 中使用 CAS。

## 1、Map/Set
### 1.1 ConcurrentHashMap/ConcurrentHashSet
底层哈希实现的同步 Map(Set)。效率高，线程安全。使用系统底层技术实现线程安全。 量级较 synchronized 低。key 和 value 不能为 null。

### 1.2 ConcurrentSkipListMap/ConcurrentSkipListSet
底层跳表(SkipList)实现的同步 Map(Set)。有序，效率比 ConcurrentHashMap 稍低。
```java
import java.util.Map;
import java.util.Random;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CountDownLatch;

public class Test_01_ConcurrentMap {

    public static void main(String[] args) {
//        final Map<String, String> map = new Hashtable<>();
         final Map<String, String> map = new ConcurrentHashMap<>();
         //ConcurrentSkipListMap跳表实现的，是排序的，最慢
//         final Map<String, String> map = new ConcurrentSkipListMap<>();
        final Random r = new Random();
        Thread[] array = new Thread[100];
        final CountDownLatch latch = new CountDownLatch(array.length);

        long begin = System.currentTimeMillis();
        for (int i = 0; i < array.length; i++) {
            array[i] = new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int j = 0; j < 100000; j++) {
                        map.put("key" + r.nextInt(100000), "value" + r.nextInt(100000));
                    }
                    latch.countDown();
                }
            });
        }
        for (Thread t : array) {
            t.start();
        }
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        long end = System.currentTimeMillis();
        System.out.println("执行时间为 ： " + (end - begin) + "毫秒！");
    }

}
```

## 2、List
### 2.1 CopyOnWriteArrayList
写时复制集合。写入效率低，读取效率高。每次写入数据，都会创建一个新的底层数组。
```java
import java.util.ArrayList;
import java.util.List;
import java.util.Random;
import java.util.concurrent.CountDownLatch;

public class Test_02_CopyOnWriteList {

    public static void main(String[] args) {
         final List<String> list = new ArrayList<>();
//         final List<String> list = new Vector<>();
//        final List<String> list = new CopyOnWriteArrayList<>();
        final Random r = new Random();
        Thread[] array = new Thread[100];
        final CountDownLatch latch = new CountDownLatch(array.length);

        long begin = System.currentTimeMillis();
        for (int i = 0; i < array.length; i++) {
            array[i] = new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int j = 0; j < 1000; j++) {
                        list.add("value" + r.nextInt(100000));
                    }
                    latch.countDown();
                }
            });
        }
        for (Thread t : array) {
            t.start();
        }
        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        long end = System.currentTimeMillis();
        System.out.println("执行时间为 ： " + (end - begin) + "毫秒！");
        System.out.println("List.size() : " + list.size());
    }

}
```

## 3、Queue
### 3.1 ConcurrentLinkedQueue
基础链表同步队列。
```java
/**
 * 并发容器 - ConcurrentLinkedQueue
 * 队列 - 链表实现的。
 */
import java.util.Queue;
import java.util.concurrent.ConcurrentLinkedQueue;

public class Test_03_ConcurrentLinkedQueue {

    public static void main(String[] args) {
        Queue<String> queue = new ConcurrentLinkedQueue<>();
        for (int i = 0; i < 10; i++) {
            queue.offer("value" + i);
        }

        System.out.println(queue);
        System.out.println(queue.size());

        // peek() -> 查看queue中的首数据
        System.out.println(queue.peek());
        System.out.println(queue.size());

        // poll() -> 获取queue中的首数据
        System.out.println(queue.poll());
        System.out.println(queue.size());
    }

}
```
### 3.2 LinkedBlockingQueue
阻塞队列，队列容量不足自动阻塞，队列容量为 0 自动阻塞。
```java
/**
 * 并发容器 - LinkedBlockingQueue
 * 阻塞容器。
 * put & take - 自动阻塞。
 * put自动阻塞， 队列容量满后，自动阻塞
 * take自动阻塞方法， 队列容量为0后，自动阻塞。
 */

import java.util.Random;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;

public class Test_04_LinkedBlockingQueue {

    final BlockingQueue<String> queue = new LinkedBlockingQueue<>();
    final Random r = new Random();

    public static void main(String[] args) {
        final Test_04_LinkedBlockingQueue t = new Test_04_LinkedBlockingQueue();

        new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    try {
                        t.queue.put("value" + t.r.nextInt(1000));
                        TimeUnit.SECONDS.sleep(1);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }, "producer").start();

        for (int i = 0; i < 3; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    while (true) {
                        try {
                            System.out.println(Thread.currentThread().getName() +
                                    " - " + t.queue.take());
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }, "consumer" + i).start();
        }
    }

}
```
### 3.3 ArrayBlockingQueue
底层数组实现的有界队列。自动阻塞。根据调用 API(add/put/offer)不同，有不同特性。
当容量不足的时候，有阻塞能力。
add 方法：在容量不足的时候，抛出异常。put 方法在容量不足的时候，阻塞等待。
offer 方法：单参数 offer 方法，不阻塞。容量不足的时候，返回 false。当前新增数据操作放弃。 三参数 offer 方法(offer(value,times,timeunit))，容量不足的时候，阻塞 times 时长(单位为 timeunit)，如果在阻塞时长内，有容量空闲，新增数据返回 true。如果阻塞时长范围内，无容量空闲，放弃新增数据，返回 false。
```java
/**
 * 并发容器 - ArrayBlockingQueue
 * 有界容器。
 */

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeUnit;

public class Test_05_ArrayBlockingQueue {

    final BlockingQueue<String> queue = new ArrayBlockingQueue<>(3);

    public static void main(String[] args) {
        final Test_05_ArrayBlockingQueue t = new Test_05_ArrayBlockingQueue();

        for (int i = 0; i < 5; i++) {
            // System.out.println("add method : " + t.queue.add("value"+i));
            /*try {
                t.queue.put("put"+i);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("put method : " + i);*/
            // System.out.println("offer method : " + t.queue.offer("value"+i));
            try {
                System.out.println("offer method : " +
                        t.queue.offer("value" + i, 1, TimeUnit.SECONDS));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        System.out.println(t.queue);
    }

}
```
### 3.4 DelayQueue
延时队列。根据比较机制，实现自定义处理顺序的队列。常用于定时任务。
如:定时关机。
```java
/**
 * 并发容器 - DelayQueue
 * 无界容器。
 */

import java.util.concurrent.BlockingQueue;
import java.util.concurrent.DelayQueue;
import java.util.concurrent.Delayed;
import java.util.concurrent.TimeUnit;

public class Test_06_DelayQueue {

    static BlockingQueue<MyTask_06> queue = new DelayQueue<>();

    public static void main(String[] args) throws InterruptedException {
        long value = System.currentTimeMillis();
        MyTask_06 task1 = new MyTask_06(value + 2000);
        MyTask_06 task2 = new MyTask_06(value + 1000);
        MyTask_06 task3 = new MyTask_06(value + 3000);
        MyTask_06 task4 = new MyTask_06(value + 2500);
        MyTask_06 task5 = new MyTask_06(value + 1500);

        queue.put(task1);
        queue.put(task2);
        queue.put(task3);
        queue.put(task4);
        queue.put(task5);

        System.out.println(queue);
        System.out.println(value);
        for (int i = 0; i < 5; i++) {
            System.out.println(queue.take());
        }
    }

}

class MyTask_06 implements Delayed {

    private long compareValue;

    public MyTask_06(long compareValue) {
        this.compareValue = compareValue;
    }

    /**
     * 比较大小。自动实现升序
     * 建议和getDelay方法配合完成。
     * 如果在DelayQueue是需要按时间完成的计划任务，必须配合getDelay方法完成。
     */
    @Override
    public int compareTo(Delayed o) {
        return (int) (this.getDelay(TimeUnit.MILLISECONDS) - o.getDelay(TimeUnit.MILLISECONDS));
    }

    /**
     * 获取计划时长的方法。
     * 根据参数TimeUnit来决定，如何返回结果值。
     */
    @Override
    public long getDelay(TimeUnit unit) {
        return unit.convert(compareValue - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
    }

    @Override
    public String toString() {
        return "Task compare value is : " + this.compareValue;
    }

}
```
### 3.5 LinkedTransferQueue
转移队列，是一个容量为 0 的队列。使用 transfer 方法，实现数据的即时处理。没有消费者，就阻塞。
```java
/**
 * 并发容器 - LinkedTransferQueue
 * 转移队列
 * add - 队列会保存数据，不做阻塞等待。
 * transfer - 是TransferQueue的特有方法。必须有消费者（take()方法的调用者）。
 * 如果没有任意线程消费数据，transfer方法阻塞。一般用于处理即时消息。
 */

import java.util.concurrent.LinkedTransferQueue;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TransferQueue;

public class Test_07_TransferQueue {

    TransferQueue<String> queue = new LinkedTransferQueue<>();

    public static void main(String[] args) {
        final Test_07_TransferQueue t = new Test_07_TransferQueue();
        
        /*new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println(Thread.currentThread().getName() + " thread begin " );
                    System.out.println(Thread.currentThread().getName() + " - " + t.queue.take());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "output thread").start();
        
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        
        try {
            t.queue.transfer("test string");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }*/

        new Thread(new Runnable() {

            @Override
            public void run() {
                try {
                    t.queue.transfer("test string");
                    // t.queue.add("test string");
                    System.out.println("add ok");
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }).start();

        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println(Thread.currentThread().getName() + " thread begin ");
                    System.out.println(Thread.currentThread().getName() + " - " + t.queue.take());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "output thread").start();

    }

}
```
### 3.6 SynchronusQueue
同步队列，是一个容量为0的队列。是一个特殊的TransferQueue。必须现有消费线程等待，才能使用的队列。
add 方法，无阻塞。若没有消费线程阻塞等待数据，则抛出异常。 
put 方法，有阻塞。若没有消费线程阻塞等待数据，则阻塞。
```java
/**
 * 并发容器 - SynchronousQueue
 */
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.SynchronousQueue;
import java.util.concurrent.TimeUnit;

public class Test_08_SynchronusQueue {

    BlockingQueue<String> queue = new SynchronousQueue<>();

    public static void main(String[] args) {
        final Test_08_SynchronusQueue t = new Test_08_SynchronusQueue();

        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println(Thread.currentThread().getName() + " thread begin ");
                    try {
                        TimeUnit.SECONDS.sleep(2);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(Thread.currentThread().getName() + " - " + t.queue.take());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "output thread").start();
        
        /*try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }*/
        // t.queue.add("test add");
        try {
            t.queue.put("test put");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println(Thread.currentThread().getName() + " queue size : " + t.queue.size());
    }

}
```