---
title: java并发编程-synchronized、volatile和threadlocal
tags: 并发编程
categories: java
date: 2022-06-26 19:20:22
---

### 一、**Synchroinzed锁**

synchroinzed锁又称为内置锁，之前提到过进程中的线程是共享进程内所有资源的，当多个线程对同一个资源执行操作时，如果不加锁可能会导致最终结果和预期结果不符，如下代码：

```java
public class SynchronizedTest {

    private long i = 0;

    public void add() {
            i++;
    }

    static class SynTest implements Runnable {

        SynchronizedTest synchronizedTest;

        public SynTest(SynchronizedTest synchronizedTest) {
            this.synchronizedTest = synchronizedTest;
        }

        @Override
        public void run() {
            for (int i = 0; i <10000 ; i++) {
                synchronizedTest.add();
            }
        }
    }

    public static void main(String[] args) {
        SynchronizedTest s = new SynchronizedTest();
        // 线程一
        new Thread(new SynTest(s)).start();
        //线程二
        new Thread(new SynTest(s)).start();
        SleepTools.second(2);
        System.out.println("结果："+s.i);
    }
}
输出：
结果：18876
```

输出的结果和预期的20000并不一样，这是因为两个线程同时对该值进行操作，每个线程执行加一操作时需要做两个步骤，首先获取到i的值，然后加一再写到内存；当线程一刚取到值但是还没写，线程二也取值，两个线程都完成加一回写时，实际上这两个线程写的是同一个值。
这时候就需要加锁，关键字synchronized 可以修饰方法或者以同步块的形式来进行使用，它主要确保多个线程在同一个时刻，只能有一个线程处于方法或者同步块中，它保证了线程对变量访问的可见性和排他性。

1. 修饰方法

```java
public synchronized void add() {
            i++;
    }
```

2. 修饰代码块

```java
public void add() {
        synchronized (this) { //this指当前实例对象
            i++;
        }
    }
```

3. 修饰成员变量

```java
 private Object o=new Object();

    public void add() {
        synchronized (o) {
            i++;
        }
    }
```

这个和修饰代码块里的this道理是一样的，都是锁的某个对象。

4. 修饰静态方法

```java
public synchronized static void add() {
            i++;
    }
```

当synchronized修饰静态方法或静态成员变量时有人说是类锁，实际上锁的虚拟机中该类所对应的class对象，本质上还是类锁。

5. 面试题
   **对象锁和类锁可以并行运行吗？**
   面试经常回问到这个问题，就如上面所说的，类锁锁的是xxx.class对象，对象锁锁的是‘对象xxx'，锁的不是同一个对象肯定是可以并行的。

6. **synchroized错误使用**，如下代码：

```java
public class ThreadStart {

    static class MyRunnable implements Runnable {

        private Integer i;

        public MyRunnable(Integer i) {
            this.i = i;
        }
        @Override
        public void run() {
            synchronized (i) {
                Thread thread=Thread.currentThread();
                System.out.println(thread.getName()+"==开始=="+i+"@"+System.identityHashCode(i));
                i++;
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(thread.getName()+"==结束=="+i+"@"+System.identityHashCode(i)); //identityHashCode方法可以理解为内存中的地址
            }
        }
    }

    public static void main(String[] args) {
        MyRunnable a=new MyRunnable(1);
        for (int i = 0; i <5 ; i++) {
            new Thread(a).start();
        }
    }
}
输出结果：
Thread-0==开始==1@351452368
Thread-1==开始==2@733316467
Thread-2==开始==3@696436475
Thread-3==开始==4@2019092735
Thread-1==结束==5@628256373
Thread-0==结束==5@628256373
Thread-3==结束==5@628256373
Thread-4==开始==5@628256373
Thread-2==结束==6@658845115
Thread-4==结束==6@658845115
```

按照预期，每个线程都对i进行加一，但实际结果thread-0起始值为1结束值为5，且地址也由351452368变为了628256373。从代码看明显是锁到了i且是同一个对象，为什么会出现这个异常？难道是没锁住？
我们把这个类反编译一下，结果如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200423214359565.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI0ODYzNzQz,size_16,color_FFFFFF,t_70)
从反编译的结果我们可以看到，当执行i++的时候实际返回的结果是Integer.valueOf(this.i.intValue() + 1);我们再去Integer类看看这个方法：

```java
public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }
```

返回的是new Integer对象，那上边的问题就能够讲的通了，当每个线程执行++操作时，i实际都指向了另一个对象，synchroinzed关键字锁的是对象，这里强调的是同一个对象，这段代码当有一个线程返回时这个锁的对象就会发生变化，所有就出现了如上的输出。

**那么这个锁应该怎么加呢？**

```java
public class ThreadStart {

    static class MyRunnable implements Runnable {

        private Integer i;

        private Object o=new Object();

        public MyRunnable(Integer i) {
            this.i = i;
        }
        @Override
        public void run() {
            synchronized (o) {
                Thread thread=Thread.currentThread();
                System.out.println(thread.getName()+"==开始=="+i+"@"+System.identityHashCode(i));
                i++;
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(thread.getName()+"==结束=="+i+"@"+System.identityHashCode(i)); //identityHashCode方法可以理解为内存中的地址
            }
        }
    }

    public static void main(String[] args) {
        MyRunnable a=new MyRunnable(1);
        for (int i = 0; i <5 ; i++) {
            new Thread(a).start();
        }
    }
}
输出：
Thread-0==开始==1@351452368
Thread-0==结束==2@439697470
Thread-4==开始==2@439697470
Thread-4==结束==3@1488227252
Thread-3==开始==3@1488227252
Thread-3==结束==4@2019092735
Thread-2==开始==4@2019092735
Thread-2==结束==5@696436475
Thread-1==开始==5@696436475
Thread-1==结束==6@733316467
```

我们锁一个不会变化的对象就可以了，检查结果也是符合预期的。

### 二、volatile

volatile是最轻量级的同步机制，一旦一个共享变量（类的成员变量、类的静态成员变量）被volatile修饰之后，那么就具备了两层语义

1. 保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。
2. 禁止进行指令重排序。

第一条比较好理解，主要是理解第二条，指令重排序的定义如下：

> 为了尽可能减少内存操作速度远慢于CPU运行速度所带来的CPU空置的影响，虚拟机会按照自己的一些规则将程序编写顺序打乱——即写在后面的代码在时间顺序上可能会先执行，而写在前面的代码会后执行——以尽可能充分地利用CPU，但最终的执行结果不变。

volatile只是保证可见性，即只是保证读一直性并不保证写只能有一个线程写，如下代码验证：

```java
public class VolatileTest {

    private volatile int i;

    public VolatileTest(int i) {
        this.i = i;
    }

    public void add() {
        i++;
    }

    static class MyRunnable implements Runnable {
        VolatileTest volatileTest;

        public MyRunnable(VolatileTest volatileTest) {
            this.volatileTest = volatileTest;
        }

        @Override
        public void run() {
            for (int i = 0; i <10000 ; i++) {
                volatileTest.add();
            }
        }
    }

    public static void main(String[] args) {
        VolatileTest v=new VolatileTest(1);
        new Thread(new MyRunnable(v)).start();
        new Thread(new MyRunnable(v)).start();

        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("i="+v.i);
    }
}
输出：
i=19695
```

所以，volatile关键字适合一写多读的场景。

### 三、ThreadLocal

ThreadLocal类的作用是在多线程编程下，保证线程间数据的隔离，实现的方法就是为每个线程维护一个变量的副本。
**1. ThreadLocal使用**

```java
    public class ThreadLocalTest {

    private static ThreadLocal<Integer> age=new ThreadLocal<Integer>(){
        @Override
        protected Integer initialValue() {
            return 1;
        }
    };

//    private static int age=1;

    static class MyRunnable implements Runnable {
        private int number;

        public MyRunnable(int number) {
            this.number = number;
        }

        @Override
        public void run() {
            Thread thread=Thread.currentThread();
            System.out.println(thread.getName()+"==start=="+age.get());
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
//            number=age.get()+number;
            age.set(number+age.get());

            System.out.println(thread.getName()+"==end=="+age.get());
        }
    }

    public static void main(String[] args) {
        ThreadLocalTest threadLocalTest=new ThreadLocalTest();

        Thread[] threads=new Thread[3];
        for (int i = 0; i <3 ; i++) {
            threads[i]=new Thread(new MyRunnable(i));
        }
        for (Thread thread : threads) {
            thread.start();
        }
    }
}
输出：
Thread-0==start==1
Thread-2==start==1
Thread-1==start==1
Thread-1==end==2
Thread-0==end==1
Thread-2==end==3
```

**2. ThreadLocal的实现解析**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200424144134690.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI0ODYzNzQz,size_16,color_FFFFFF,t_70)

声明一个ThreadLocal类型的属性，每个线程都会维护自己的一个ThreadLocalMap属性，查看Thread原密会发现这个属性![在这里插入图片描述](https://img-blog.csdnimg.cn/20200424144340140.png)
可看到该属性是专门为ThreadLocal设计的，而ThreadLocalMap又是ThreadLocal的内部类，查看ThreadLocal源码：![在这里插入图片描述](https://img-blog.csdnimg.cn/2020042414455461.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI0ODYzNzQz,size_16,color_FFFFFF,t_70)
可以看到有个 Entry 内部静态类，它继承了 WeakReference，总之它记录了两个信息，一个是 ThreadLocal<?>类型，一个是 Object 类型的值。getEntry 方法则是获取某个 ThreadLocal 对应的值，set 方法就是更新或赋值相应的 ThreadLocal
对应的值。
ThreadLocal的值获取流程就是，先获取当前线程的ThreadLocalMap属性，然后根据当前ThreadLocal的实例获取map中的Entry对象，即获取了值。

**3. ThreadLocal使用不当可能造成内存溢出**
先看如下代码：

```java
public class ThreadLocalOOMl implements Runnable {

    private static final int TASK_MAX_LOOP=500;

    public static ThreadPoolExecutor threadPoolExecutor=new ThreadPoolExecutor(5,5,1,
            TimeUnit.MINUTES,new LinkedBlockingQueue<>());

    @Override
    public void run() {
        new LocalVariable(); //执行创建数组
    }

    static class LocalVariable{
        private byte[] b=new byte[1024*1024*5]; //创建一个5M的数组
    }

    public static void main(String[] args) {
        for (int i = 0; i <TASK_MAX_LOOP ; i++) {
            threadPoolExecutor.execute(new ThreadLocalOOMl());
            System.out.println("not use threadLocal!");
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        System.out.println("thread excute over!");
    }
}

```

使用jdk自带的jvm监控工具（位置在bin/jvisualvm.exe），查看堆的使用大小如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200424185246548.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI0ODYzNzQz,size_16,color_FFFFFF,t_70#pic_center)
我们线程池的大小是5，也就是最多5个线程同时运行，占用的堆内存基本维持在25M以下这是正常的。有人可能会问每个线程new一个5M的数组为什么只有25M？看下代码new ThreadLocalOOMl()，我们每次只是在堆上new了一个对象，该对象并没有被引用，所以该对象会被回收，所以堆的内存大概只有25M。

使用ThreadLocal代码如下：

```java
public class ThreadLocalOOMl implements Runnable {

    private static final int TASK_MAX_LOOP=500;

    public static ThreadPoolExecutor threadPoolExecutor=new ThreadPoolExecutor(5,5,1,
            TimeUnit.MINUTES,new LinkedBlockingQueue<>());

    public static ThreadLocal<LocalVariable> localVariable=new ThreadLocal<>();

    @Override
    public void run() {
        localVariable.set(new LocalVariable()); //每个线程维护一个threadLocal副本
    }

    static class LocalVariable{
        private byte[] b=new byte[1024*1024*5]; //创建一个5M的数组
    }


    public static void main(String[] args) {
        for (int i = 0; i <TASK_MAX_LOOP ; i++) {
            threadPoolExecutor.execute(new ThreadLocalOOMl());
            System.out.println("use threadLocal!");
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        System.out.println("thread excute over!");
    }
}

```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200424190850316.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI0ODYzNzQz,size_16,color_FFFFFF,t_70#pic_center)
我们的预期结果应该是和不使用threadLocal一致大概25M左右，实际上最高已经来到250左右，出现了内存溢出。这是为什么呢？接下来分析

分析之前先说以下java里的几种引用：

1. 强引用
   就是指在程序代码之中普遍存在的，类似“Object obj=new Object（）”这类的引用，只要强引用还存在，垃圾收集器永远不会回收掉被引用的对象实例。
2. 软引用
   用来描述一些还有用但并非必需的对象。对于软引用关联着的对象，在系统将要发生内存溢出异常之前，将会把这些对象实例列进回收范围之中进行第二次回收。如果这次回收还没有足够的内存，才会抛出内存溢出异常。在 JDK1.2 之后，提供了 SoftReference 类来实现软引用。
3. 弱引用
   用来描述非必需对象的，但是它的强度比软引用更弱一些，被弱引用关联的对象实例只能生存到下一次垃圾收集发生之前。当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象实例。在 JDK 1.2 之后，提供了WeakReference 类来实现弱引用。
4. 虚引用也称为幽灵引用或者幻影引用
   它是最弱的一种引用关系。一个对象实例是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。为一个对象设置虚引用关联的唯一目的就是能在这个对象实例被收集器回收时收到一个系统通知。在 JDK 1.2 之后，提供了PhantomReference 类来实现虚引用。

再来梳理以下ThreadLocal这个类，每个线程维护了一个ThreadLocalMap对象，它的key是ThreadLocal对象，值是Entry数组，真正存储数据的是Enter，
查看源码我们会发现Entry对ThreadLocal引用是弱引用
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200424192244326.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI0ODYzNzQz,size_16,color_FFFFFF,t_70)
前边说过当发生内存回收的时候，ThreadLocal会被回收，所有使用ThreadLocal后整个调用关系可用如下图描述：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200424192505955.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI0ODYzNzQz,size_16,color_FFFFFF,t_70)
当发生内存回收时，ThradLocal对象被回收，此时该对象所对应的Entry的value将会永远不会被访问，而Entry对该值是强引用，所以该value无法被回收这就造成了内存泄露。除非当线程结束时，下方的调用链就是被回收，但是我们这里使用了线程池，线程不会结束。。。那么怎么办呢？
其实只要在你使用完这个值，主动把这个值释放掉就可以了，代码修改比较简单：

```java
  @Override
    public void run() {
        localVariable.set(new LocalVariable()); //每个线程维护一个threadLocal副本
        System.out.println("我获取到了值，可以释放了...");
        localVariable.remove();
    }
```

使用完调用remove()方法，就会释放该值，可以查看remove的源码，发现最终会调到expungeStaleEntry()释放没用的Entry。查看堆的大小就正常了
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200424193506308.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI0ODYzNzQz,size_16,color_FFFFFF,t_70#pic_center)
**所以，在使用线程池和ThreadLocal的时候要注意这个问题，在使用完值后一定要记得释放。**

### 四、总结

synchronized解决了多线程编程下的数据共享问题，而ThreadLocal则是实现了线程间的隔离。但如果使用的不好也能造成其他的问题，一定要牢记。
