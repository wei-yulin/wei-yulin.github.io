---
title: Java并发编程-AQS同步器源码解析
tags: 并发编程
categories: java
date: 2022-06-27 13:57:42
---

### 一、CLH队列锁
学习AQS之前先来了解下CLH队列锁，即 Craig, Landin, and Hagersten (CLH) locks。顾名思义，该锁维护了一个队列，每个节点代表一个线程，结构图如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200502170437804.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI0ODYzNzQz,size_16,color_FFFFFF,t_70)
当一个线程想要获取锁，首先把自己包装成一个队列节点，该节点需要记录前驱节点和一个是否需要获取锁的标识blocked默认值为true的，然后在前驱节点的blocked属性上进行自旋，知道前驱节点释放锁blocked为false时，标识该线程可以拿到锁并停止自旋，该线程执行完业务逻辑后释放锁blocked变为false，下一个节点拿到锁重复相同步骤，直到队列为空。

AQS(AbstractQueuedSynchronizer)是java中CLH锁的一种变体实现，它是用来构建锁或者其他同步组件的基础框架，它使用了一个 int 成员变量表示同步状态，通过内置的 FIFO 队列来完成资源获取线程的排队工作。常用的ReentrantLock和ReentrantReadWriteLock锁都是基于AQS来实现了。
### 一、AQS使用的设计模式
1. **模板设计模式:**
什么是模板设计模式？定义了一个操作中的算法的骨架，而将部分步骤的实现在子类中完成。模板方法模式使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤。
AQS中获取同步状态的方法，例如acquire():

```java
	public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

	protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
```
该方法在AQS中并没有实现，而是交给子类去实现，而AQS的主流程中又依赖这个方法，所以这是典型的模板设计方法。
在AQS中使用模板设计模式的有如下方法，在实现自己的同步器的时候需要重写对应的方法，这些模板方法基本上分为 3 类：独占式获取与释放同步状态、共享式获取与释放、同步状态和查询同步队列中的等待线程情况。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200502160342612.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI0ODYzNzQz,size_16,color_FFFFFF,t_70)

### 二、独占式同步状态获取与释放
#### 1、同步状态获取

```java
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
首先调用tryAcquire方法而且是根据该方法的返回接过来确定是否接着往下走，因为是独占式的获取，所以拿锁的动作要么成功要么失败。如果返回true则认为线程成功获取同步状态，继续执行业务代码。如果获取失败则，调用acquireQueued(addWaiter(Node.EXCLUSIVE), arg)
**先来看下addWaiter()**

```java
private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }

private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```
该方法接收一个mode参数，该参数在AQS的内部类中有定义，一个是标识共享模式等待，一个是独占式。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200502171426380.png)
该方法先将线程封装成一个节点，目的是将该节点设置为当前队列的尾节点，先尝试设置一次，如果成功就返回设置成功。如果失败就进入enq()方法开始无限循环，直到把自己设置为尾节点。
**接下来是acquireQueued**

```java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

  private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
先拿到该节点的前驱节点并判断是否是head节点，如果是则再次尝试获取同步状态成功则把自己置为头节点，并将头节点指向null使之能被gc回收，然后返回；失败则继续往下走。
parkAndCheckInterrupt（）该方法则是调用LockSupport的park的方法。这里先说一下LockSupport，LockSupport 定义了一组以 park 开头的方法用来阻塞当前线程，以及unpark(Thread thread)方法来唤醒一个被阻塞的线程，这些方法提供了最基本的线程阻塞和唤醒功能，而 LockSupport 也成为构建同步组件的基础工具。
所以如果线程没有获取到同步状态，这该线程会被放在FIFO队列里并且是一个阻塞状态。那么什么时候该线程能够被唤起呢？

#### 2、同步状态释放
当线程的业务代码执行完，需要释放同步状态，独占式的释放调用的是release方法

```java
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```
首先调用tryRelease（）释放锁，失败则返回，成功则释放它的下一个被阻塞的节点。![在这里插入图片描述](https://img-blog.csdnimg.cn/20200502184609765.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI0ODYzNzQz,size_16,color_FFFFFF,t_70)
这段代码可能有人会看的比较晕，其实是它先拿到下一个节点，先判断节点的状态。AQS类有个Node内部类，这里定义了Node的四种状态：

> CANCELLED：值为 1，表示线程的获锁请求已经“取消” SIGNAL：值为-1，表示该线程一切都准备好了,就等待锁空闲出来给我
> CONDITION：值为-2，表示线程等待某一个条件（Condition）被满足
> PROPAGATE：值为-3，当线程处在“SHARED”模式时，该字段才会被使用 上 初始化 Node 对象时，默认为 0

所以当节点的状态大于0时只有删除状态，而唤醒的节点不应该是已经删除的，而for (Node t = tail; t != null && t != node; t = t.prev)明显看出，下一个节点是从队列的尾开始遍历而不是拿删除节点的下一个节点，至于为什么？ 猜测可能的原因是导致节点删除的原因主要是超时，而既然先请求的节点都删除了，那么删除节点的下一个节点也大概率是删除的，所以从尾部遍历。找到未删除的节点，然后调用unPark唤醒。

### 三、共享式的同步状态获取与释放
#### 1、同步状态获取
我们先看下共享式的释放代码

```java
public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }

private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
tryAcquireShared(arg) < 0，判断是否获取成功不再是Boolean值，这是因为共享式的获取可能有多个线程同时获取到同步状态。可以认为tryAcquireShared返回大于0的值时成功拿到，而同步状态被拿完了则进入等待队列。进入等待队列时，判断前一个节点是否是head节点，如果是则再尝试获取一次锁，如果没有获取到则调用parkAndCheckInterrupt进入阻塞状态，等待被唤起。
此时要是成功拿到了，它会认为还有空闲的可以拿，则会调用setHeadAndPropagate唤起它后面的节点去获取同步状态，如果获取成功接着传播，失败则再次进入阻塞状态等待唤起。

#### 2.  共享式的释放

```java
 public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }

private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }
```
首先调用tryReleaseShared（）检查是否可以释放，false则直接返回释放失败，true则会唤醒下一个节点。由于时共享式的释放，所以可能你还没有唤醒下一节点，已经被其他线程唤醒，这时候会再循环一次直到唤醒下一个节点或者没有可被唤醒的节点退出。

### 四、超时获取与释放
无论是独占式还是共享式的超时同步状态与获取其实都是一样的，调用LockSupport.park式传入一个超时时间，超过时间未被唤醒则中断返回；如果未超时被唤醒则重新获取，未获取到重新计算超时时间继续阻塞。

```java
private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout;
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor(); //获取前驱节点
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
                nanosTimeout = deadline - System.nanoTime();  //重新计算
                if (nanosTimeout <= 0L)
                    return false;
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout); //超时时间
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

### 五、实现独占式锁

```java
public class MyExclusiveLock implements Lock {

    // 静态内部类，自定义同步器
    class Sync extends AbstractQueuedSynchronizer {

        @Override
        protected boolean tryAcquire(int arg) {
            Thread thread = Thread.currentThread();
            if (compareAndSetState(0, arg)) {
                setExclusiveOwnerThread(thread);  //表示当前线程拿到了锁
                return true;
            }
            return false;
        }

        @Override
        protected boolean tryRelease(int arg) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }
    }

    Sync sync=new Sync();

    @Override
    public void lock() {
        System.out.println(Thread.currentThread().getName()+" ready get lock");
        sync.acquire(1);
        System.out.println(Thread.currentThread().getName()+"  allready got lock");
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {

    }

    @Override
    public boolean tryLock() {
        return false;
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return false;
    }

    @Override
    public void unlock() {
        System.out.println(Thread.currentThread().getName()+" release lock");
        sync.release(1);
    }

    @Override
    public Condition newCondition() {
        return null;
    }
}

```
测试代码：

```java
public class TestMyLock {

    public static void main(String[] args) {

        long start=System.currentTimeMillis();
        MyExclusiveLock lock = new MyExclusiveLock();
        Thread[] threads=new Thread[5];
        for (int i = 0; i < 5; i++) {
            Thread thread=new Thread(() -> {
                lock.lock();
                ms(2000);
                lock.unlock();
            });
            threads[i]=thread;
        }

        for (Thread thread : threads) {
            thread.start();
        }
        for (Thread thread : threads) {
            try {
                thread.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        System.out.println("耗时: "+(System.currentTimeMillis()-start)+"ms");
    }
}
输出：
Thread-0 ready get lock
Thread-2 ready get lock
Thread-2  allready got lock
Thread-1 ready get lock
Thread-3 ready get lock
Thread-4 ready get lock
Thread-2 release lock
Thread-1  allready got lock
Thread-1 release lock
Thread-0  allready got lock
Thread-0 release lock
Thread-3  allready got lock
Thread-3 release lock
Thread-4  allready got lock
Thread-4 release lock
耗时: 10300ms
```

### 六、实现共享式锁
共享锁代码：

```java
public class MySharedLock implements Lock {

    class Sync extends AbstractQueuedSynchronizer {

        private int count; //定义可同时获取锁的线程数

        public Sync(int count) {
            this.count = count;
            setState(count);
        }

        @Override
        protected int tryAcquireShared(int arg) {
            for (; ; ) {
                int state = getState();
                int newState = state - arg;
                if (newState < 0 || compareAndSetState(state, newState)) {
                    return newState;
                }
            }
        }

        @Override
        protected boolean tryReleaseShared(int arg) {
            for (; ; ) {
                int state = getState();
                int newState = state + arg;
                if (newState > count) {
                    return false;
                } else {
                    return compareAndSetState(state, newState);
                }

            }
        }

    }


    Sync sync=new Sync(3); //同时支持3个线程获取到锁
    @Override
    public void lock() {
        System.out.println(Thread.currentThread().getName()+"ready get lock");
        sync.acquireShared(1);
        System.out.println(Thread.currentThread().getName()+"allready got lock");
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {

    }

    @Override
    public boolean tryLock() {
        return false;
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return false;
    }

    @Override
    public void unlock() {
        System.out.println(Thread.currentThread().getName()+"release lock");
        sync.releaseShared(1);
    }

    @Override
    public Condition newCondition() {
        return null;
    }
}

```

测试代码：
```java
public class TestMySharedLock {

    public static void main(String[] args) {
        long start=System.currentTimeMillis();
        MySharedLock lock = new MySharedLock();
        Thread[] threads=new Thread[9];
        for (int i = 0; i < 9; i++) {
            Thread thread=new Thread(() -> {
                lock.lock();
                ms(2000);
                lock.unlock();
            });
            threads[i]=thread;
        }

        for (Thread thread : threads) {
            thread.start();
        }
        for (Thread thread : threads) {
            try {
                thread.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        System.out.println("耗时: "+(System.currentTimeMillis()-start)+"ms");
    }

}

输出：
Thread-0ready get lock
Thread-0allready got lock
Thread-1ready get lock
Thread-1allready got lock
Thread-2ready get lock
Thread-2allready got lock
Thread-3ready get lock
Thread-4ready get lock
Thread-5ready get lock
Thread-7ready get lock
Thread-6ready get lock
Thread-8ready get lock
Thread-1release lock
Thread-3allready got lock
Thread-2release lock
Thread-4allready got lock
Thread-0release lock
Thread-5allready got lock
Thread-3release lock
Thread-7allready got lock
Thread-4release lock
Thread-6allready got lock
Thread-5release lock
Thread-8allready got lock
Thread-7release lock
Thread-6release lock
Thread-8release lock
耗时: 6091ms
```
共9个线程每个线程休眠2秒钟，我们的锁定义的可同时支持3个线程拿到锁，所以耗时6091ms可证明每次确实有三个线程获取到锁。

### 七、总结
在获取同步状态时，同步器维护一个同步队列，获取状态失败的线程都会被加入到队列中并在队列中进行自旋；移出队列（或停止自旋）的条件是前驱节点为头节点且成功获取了同步状态。在释放同步状态时，同步器调用 tryRelease(int arg)方法释放同步状态，然后唤醒 head 指向节点的后继节点
