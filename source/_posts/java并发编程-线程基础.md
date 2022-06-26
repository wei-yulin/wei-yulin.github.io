---
title: java并发编程-线程基础
tags: 并发编程
categories: java
date: 2022-06-26 17:57:23
---

### 1.进程与线程的区别

 1. **进程是程序运行资源分配的最小单位**，也就是说操作系统封是以进程为单位进行分配资源的，资源包括cpu时间片、内存、磁盘io等；进程与进程之间是相互独立的，一个应用程序可以理解为一个进程，当你打开微信又打开支付宝的时候这两个应用都可以正常工作，所以说进程之间的独立的。进程又可以分为系统进程和用户进程，比如当你按下电脑卡机键，你就可以打开桌面，这是系统启动的进程；而你打开浏览器，这是你自己启动的进程，就是用户进程。
 2.  **线程是cpu调度的最小单位**，线程必须依赖于进程存在，一个进程内可以有线程，所有线程共享进程的资源，线程负责执行进程的各个功能，线程本身占用很少的资源（如程序计数器）

### 2.java中有几种新启线程方式
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;对于这个问题大多数人可能会想到3种，继承Thread、实现Runnable接口、实现Callable接口，其实java的官方给出的答案是2种，它认为实现Runnable接口、实现Callable接口是一种方式。

> **There are two ways to create a new thread of execution.	--Thread类第73行注释**

两种启动线程的方式如下：

```java
public class ThreadStart {

    static class MyThread extends Thread {
        @Override
        public void run() {
            System.out.println("myThread start....");
        }
    }
    
    static class MyRunnable implements Runnable {
        @Override
        public void run() {
            System.out.println("myRunnable start...");
        }
    }

    public static void main(String[] args) {
        MyThread a=new MyThread();
        a.start();

        MyRunnable b=new MyRunnable();
        new Thread(b).start();
    }

	输出：
		myThread start....
		myRunnable start...
}
```
### 3.对于start()方法和run()方法的理解
1.	run()方法和线程的启动与否没有关系，它是具体执行业务逻辑的功能，就是一个成员方法，实例化类后可以直接调用。如下代码可以证明：

```java
	public class ThreadStart {

    static class MyRunnable implements Runnable {
        @Override
        public void run() {
            System.out.println("myRunnable start...");
        }
    }

    public static void main(String[] args) {

        MyRunnable b=new MyRunnable();
        b.run();
    }

}
	输出
	myRunnable start...
```
2. start()方法是真正创建线程的方法，查看源码会发现它调用的start0()方法，该方法的定义*private native void start0();*实际上这个方法调用的jvm里由C语言写的方法去调操作系统的api完成线程的创建。

### 4.如何让线程安全停止
1. **线程停止的方式**	

    * 执行完run()方法，程序自然结束

    - 程序抛出未处理的异常，导致终止

2. **线程的暂停、恢复和停止操作（不安全）**

    线程的暂停、恢复和停止操作对应的API分别为~~suspend()~~ 、~~resume()~~ 、~~stop~~ 。查看源码发现这些方法已经都被标注为过期不建议使用，因为这些方法都比较野蛮，不会给线程释放资源的时间，如果线程持有锁可能会导致死锁的出现。

3. **如何安全的终止线程**
  **interrupt()方法**

  Thread提供了interrupt()方法，该方法的作用是另启动一个线程给当前线程打标签（标签的值有ture和false两）；因为java中的线程是协作式的并非抢占式的，所以线程被打上中断的标签后可以不做理会。如下代码验证：


```java
public class InterruptTest {

    static class MyThread extends Thread {
        @Override
        public void run() {
            String threadName=Thread.currentThread().getName();
            System.out.println(threadName+" interrupt flag: "+isInterrupted());
            while (true) {
                System.out.println(threadName+" is running");
                System.out.println(threadName+" interrupt flag: "+isInterrupted());
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        MyThread thread=new MyThread();
        thread.start();
        Thread.sleep(1000);
        thread.interrupt();
    }
}
输出（截取一部分）：
Thread-0 interrupt flag: false
Thread-0 is running
Thread-0 interrupt flag: false
Thread-0 is running
Thread-0 interrupt flag: false
Thread-0 is running
Thread-0 interrupt flag: true
Thread-0 is running
Thread-0 interrupt flag: true
Thread-0 is running
Thread-0 interrupt flag: true
```
根据输出结果可以看到，即使调用了interrupt()方法，线程依然可以继续执行。

**监控标签值控制程序执行**
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;例如我在判断标签值之前就释放该线程的所占有的锁或者资源，这样即使线程中断也不会再持有锁和资源，不会发生死锁的情况也不会造成资源的浪费

```java
public class InterruptTest {

    static class MyThread extends Thread {
        @Override
        public void run() {
            String threadName=Thread.currentThread().getName();
            System.out.println(threadName+" interrupt flag: "+isInterrupted());
//            while (true) {
            while (!isInterrupted()) {
                System.out.println(threadName+" is running");
                System.out.println(threadName+" inner interrupt flag: "+isInterrupted());
            }
            System.out.println(threadName+" interrupt flag: "+isInterrupted());
        }
    }

    public static void main(String[] args) throws InterruptedException {
        MyThread thread=new MyThread();
        thread.start();
        Thread.sleep(3);
        thread.interrupt();
    }
	
输出：
......
Thread-0 is running
Thread-0 inner interrupt flag: false
Thread-0 is running
Thread-0 inner interrupt flag: false
Thread-0 is running
Thread-0 inner interrupt flag: false
Thread-0 is running
Thread-0 inner interrupt flag: true
Thread-0 interrupt flag: true

Process finished with exit code 0
```
**isInterrupted()和Thread.interrupted的区别**

前者只是返回标志位的值，后者也是返回标志位的值并且把标志位的值改为false，如下代码验证：

```java
public class InterruptTest {

    static class MyThread extends Thread {
        @Override
        public void run() {
            String threadName=Thread.currentThread().getName();
            System.out.println(threadName+" interrupt flag: "+isInterrupted());
//            while (true) {
//            while (!isInterrupted()) {
            while (!interrupted()) {
                System.out.println(threadName+" is running");
                System.out.println(threadName+" inner interrupt flag: "+isInterrupted());
            }
            System.out.println(threadName+" interrupt flag: "+isInterrupted());
        }
    }

    public static void main(String[] args) throws InterruptedException {
        MyThread thread=new MyThread();
        thread.start();
        Thread.sleep(3);
        thread.interrupt();
    }
}
输出：
......
Thread-0 is running
Thread-0 inner interrupt flag: false
Thread-0 is running
Thread-0 inner interrupt flag: false
Thread-0 is running
Thread-0 inner interrupt flag: false
Thread-0 interrupt flag: false

Process finished with exit code 0
```
### 5.yield()、priority()和join()方法
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;为什么会把这几个方法放到一起说，因为面试的时候经常会围绕这几个方法问一些线程执行顺序的问题。
**1. yield方法**
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;yield方法是让出当前线程的cpu时间片，使线程由运行状态变为就绪状态。注意这里并不会释放资源，让出的时间也是不可以指定的。**就绪状态的线程都有可能被cpu选择**，举个例子：皇上有三个妃子A、B、C在等着侍寝，皇上叫A进去侍寝，中途A想到和BC共苦的日子不容易，所以就想着共同致富，于是A说自己要上厕所就出去了，但是皇上不乐意了，我乃九五至尊岂容你撒野，于是又把A叫进去了，A接着侍寝......
**2. setPriority方法**
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;priority方法是设置线程执行的优先级，根据字面理解可能会误认为优先级高的一定会在优先级低的线程之前执行，实际上不是这样的，优先级高只不过是被cpu选择的概率会大一点。验证代码如下：

```java
public class YieldTest {

    static class MyThread extends Thread {
        
        public MyThread(String name) {
            super(name);
        }

        @Override
        public void run() {
            for (int i = 0; i < 1000; i++) {
                System.out.println(Thread.currentThread().getName() + " 开始运行..."+i);
                if (i == 600) {
                    yield();
                }
            }
        }
    }

    public static void main(String[] args) {
        MyThread a = new MyThread("线程一");
        a.setPriority(1);
        MyThread b = new MyThread("线程二");
        b.setPriority(10);
        a.start();
        b.start();
    }
}
输出：
...
线程二 开始运行...600
线程一 开始运行...17
线程一 开始运行...18
线程二 开始运行...601
...
线程二 开始运行...834
线程一 开始运行...19
线程二 开始运行...835
```
从代码的输出结果看，线程二的优先级高确实执行的比较快以为被cpu选择的概率大，但cpu并不是100%选择线程二，也会选择线程一执行。所以这就验证上述结论是正确的。
**3. join方法**
join()方法是将交替执行的线程设置为顺序执行，比如在A线程中调用B的join方法，则A会等B执行完再继续执行。代码如下：

```java
public class JoinTest extends Thread {

    static class Goddless implements Runnable {

        private Thread thread;

        public Goddless(Thread thread) {
            this.thread = thread;
        }

        @Override
        public void run() {
            System.out.println("goddless 开始打饭...");
            try {
                if (thread != null) {
                    thread.join();
                }
            } catch (InterruptedException e) {
            }
            SleepTools.second(2);
            System.out.println("goddles 打饭结束...");
        }
    }

    static class GoddlessBoyFriend extends Thread {

        @Override
        public void run() {
            System.out.println("goddless boyfirend 开始打饭...");
            SleepTools.second(2);
            System.out.println("goddless boyfriend 打饭结束...");
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread current=Thread.currentThread();

        Thread gbf=new GoddlessBoyFriend();

        Goddless goddless=new Goddless(gbf);
        Thread g=new Thread(goddless);
        g.start();
        gbf.start();
        System.out.println(Thread.currentThread().getName()+" 你开始打饭...");
        g.join();
        SleepTools.second(2);
        System.out.println(Thread.currentThread().getName()+" 你打饭结束...");
    }
}
```

> 程序执行流程：
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;你正在打饭刚好看见女神来打饭，你让女神插队排在你的前面，正当你狡猾的露出笑容时女神看到的男朋友并让他排在女神前边打饭，女神男朋友打完饭接着女神打饭女神也打完饭两人手牵手走到了唯一的一个空座位坐下有说有笑的开始吃饭。终于轮到了你打饭，阿姨说就剩半个馒头拿走送你了...

### 6.守护线程
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Daemon（守护）线程是一种支持型线程，因为它主要被用作程序中后台调 度以及支持性工作。这意味着，当一个 Java 虚拟机中不存在非 Daemon 线程的 时候，Java 虚拟机将会退出。可以通过调用 Thread.setDaemon(true)将线程设置 为 Daemon 线程。我们一般用不上，比如垃圾回收线程就是 Daemon 线程。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Daemon 线程被用作完成支持性工作，但是在 Java 虚拟机退出时 Daemon 线 程中的 finally 块并不一定会执行。在构建 Daemon 线程时，不能依靠 finally 块中 的内容来确保执行关闭或清理资源的逻辑。 

### 7.写给自己
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我是一个菜鸟，这是我写的第一篇博客大概用了5个小时，从现在开始我会用博客记录自己学到的知识，也希望自己能够坚持下来，记录自己的成长。努力是为了给自己更多的选择！加油！
