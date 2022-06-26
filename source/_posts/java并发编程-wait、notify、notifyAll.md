---
title: java并发编程-wait、notify、notifyAll
tags: 并发编程
categories: java
date: 2022-06-26 23:01:50
---

**首先说以下这个几个方法归属，这几个方法并不是Thread类的方法，而是Object的方法，只是在多线程编程中实现线程间同步用到而已。**

### 一、方法的作用
#### 1.wait()方法
如果某个线程调用的对象的wait()方法，那么该线程会进入到该对象的等待池中，等待池中的线程不会去竞争该对象的锁。
#### 2.notify/notifyAll方法
当某个线程调用了对象的notify方法，那么会在该对象的等待池中唤醒一个线程进入到锁池中，锁池中的线程是可以参与竞争对象的锁的。notifyAll方法则是唤醒所有等待池中的方法进入到等待池中。当该线程执行完run方法后释放对象的锁，则所有锁池中的线程开始竞争锁，其他未竞争到锁的线程仍旧会在锁池中等待下一次锁释放。

### 二、wait\notify编程的标准范式

```java
Object o=new Object();
synchronized(o){
	while(condition 不满足条件){
		o.wait();
	}
	do sth;
}

synchronized(o){
	change condition;
	o.notify()/o.notifyAll();
}
```

### 三、面试题

**问题：让两个线程交替打印0-100**

首先我们定义一个打印机：

```java
public class OrderPrint {

    public int number;

    public OrderPrint(int number) {
        this.number = number;
    }

    public synchronized void Oushu() {
        Thread thread = Thread.currentThread();
        while (number < 101 && thread.isInterrupted() == false) {
            if (number % 2 == 0) {
                System.out.println(thread.getName() + "print: " + number);
                number++;
                SleepTools.ms(100);
                notify();
                if (number == 101) { // 判断是否已打印最后一个数字，如果是则通知线程你该关闭了
                    thread.interrupt();
                }
            } else {
                try {
                    wait();
                } catch (InterruptedException e) {
//                    e.printStackTrace();
                }
            }
        }
        System.out.println("偶数打印完毕, "+thread.getName()+"==结束");
    }

    public synchronized void Jishu() {
        Thread thread = Thread.currentThread();
        while (number < 101 && thread.isInterrupted() == false) {
            if (number % 2 == 1) {
                System.out.println(thread.getName() + "print: " + number);
                number++;
                SleepTools.ms(100);
                notify();
                if (number == 99) {   // 判断是否已打印最后一个数字，如果是则通知线程你该关闭了
                    thread.interrupt();
                }
            } else {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        System.out.println("奇数打印完毕, " + thread.getName() + "==结束");

    }
}
```
再定义两个线程，分别打印奇数和偶数：

```java
public class OrderPrintTest {
    public static OrderPrint orderPrint=new OrderPrint(0);

    static class OushuRunnable implements Runnable {
        @Override
        public void run() {
            orderPrint.Oushu();
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
            }
        }
    }

    static class JishuRunnable implements Runnable {
        @Override
        public void run() {
            orderPrint.Jishu();
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
            }
        }
    }

    public static void main(String[] args) {
        new Thread(new JishuRunnable()).start();

        new Thread(new OushuRunnable()).start();
    }
}

输出：
。。。
Thread-1print: 94
Thread-0print: 95
Thread-1print: 96
Thread-0print: 97
Thread-1print: 98
Thread-0print: 99
Thread-1print: 100
偶数打印完毕, Thread-1==结束
奇数打印完毕, Thread-0==结束
```
**代码分析**：
启动两个线程一起竞争打印机的锁，其中一个线程获取到锁后执行打印前先判断当前数字是否是我的业务范围，如果是：打印这个数字，判断是不是我需要打印的最后一个数字，接着进入wait()状态。

1. 为社么要判断是不是最后一个数字？
假设没有这个判断，奇数打印完99之后number=100，然后进入wait状态；偶数打印完100之后number=101，通知奇数线程启动，奇数判断不符合条件结束线程。此时偶数线程还处在wait状态，导致线程不能被唤醒，一直处于wait状态。这里关于interrupt的用法可以参考之前的文章。
2. 线程执行notify之后可能还会拿到锁吗？
亲身实验过了，以为调用notify之后并没有执行对象的wait方法，该方法执行完任务之后还是会进入到锁池和被唤醒的线程一起去争夺锁。即使再拿到锁，不符合要求会调用对象的wait方法，使线程再进入到wait状态。
