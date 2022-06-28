---
title: java并发编程-可重入锁、公平锁与非公平锁、读写锁
tags: 锁
categories: java
date: 2022-06-28 13:11:10
---

### 一、可重入锁
1. 定义
可重入锁，也叫做递归锁，指的是在同一线程内，外层函数获得锁之后，内层递归函数仍然可以获取到该锁。 换一种说法：同一个线程再次进入同步代码时，可以使用自己已获取到的锁。 防止在同一线程中多次获取锁而导致死锁发生。

2. 在ReentrantLock中的实现

**获取锁**
```java
final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            //第一次拿到锁，设置state为acquire
            if (c == 0) { 
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            // 该线程再次进入锁，设置state为state+acquire
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
如果对同步状态获取还有疑问的可以参考上一篇文章[AQS同步器源码解析](https://blog.csdn.net/qq_24863743/article/details/105891274)，由以上代码我们可以看出当线第一次拿到锁的时候即if (c == 0) 为true时，设置state=acruire；当再次进入该代码块时，判断如果是同一个线程则对state做一个累加，返回true即认为获取锁成功。这里可能有疑问的是为什么要做累加而不是直接返回true?其实这个操作是为了之后的释放锁，这里累加操作的时候也相当于记录一个进入的次数，以便释放的时候直到何时释放完成。

**释放锁**

```java
protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) { // 如果state为0则认为释放完成
                free = true;
                setExclusiveOwnerThread(null); //标识自己已不再持有锁
            }
            setState(c); // state如果不为0，认为还为释放完成，该线程还占有锁
            return free;
        }
```
释放锁的时候，先判断同步状态是否为0，如果是则认为锁释放完成并把标识自己已不再持有该锁；如果否则认为还为释放完成，只是简单的设置状态。

### 二、公平锁与非公平锁
1. 定义
公平锁（Fair）：加锁前检查是否有排队等待的线程，优先排队等待的线程，先来先得
非公平锁（Nonfair）：加锁时不考虑排队等待问题，直接尝试获取锁，获取不到自动到队尾等待
通常非公平锁的性能高于公平锁，公平锁需要等待线程唤醒，唤醒线程需要一定的开销；而非公平锁很有可能会在唤醒线程过程中拿到锁执行完任务并释放了锁；此时被唤醒的线程继续拿锁做其他的事，所以非公平锁的效率要高于公平锁。

2. ReentrantLock中公平锁和非公平锁的实现
**非公平锁**

```java
final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
**公平锁**

```java
protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
对比公平锁和非公平锁的实现，我们会发现差别就在于公平锁（if(!hasQueuedPredecessors() &&compareAndSetState(0, acquires))）获取同步状态时先去判定等待队列是否有正在排队的线程，有则直接去队尾排队，无则使用CAS指令去修改同步状态。非公平锁（if (compareAndSetState(0, acquires))）直接就去修改同步状态，不管你队列中是否有排队的节点。

### 三、读写锁（ReentrantReadWriteLock 的实现）

> 写独占，读共享，读写互斥

这句话我觉得总结得很精辟，读写锁非常适合读多写少的场景，而互联网企业的多数业务都是这种场景。

1. 读写状态的设计
读写锁同样依赖自定义同步器来实现同步功能，而读写状态就是其同步器的
同步状态。回想 ReentrantLock 中自定义同步器的实现，同步状态表示锁被一个线程重复获取的次数，而读写锁的自定义同步器需要在同步状态（一个整型变量）上维护多个读线程和一个写线程的状态，使得该状态的设计成为读写锁实现的关键。如果在一个整型变量上维护多种状态，就一定需要“按位切割使用”这个变量，读写锁将变量切分成了两个部分，高 16 位表示读，低 16 位表示写，读写锁是如何迅速确定读和写各自的状态呢？答案是通过位运算。假设当前同步状态值为 S，写状态等于 S&0x0000FFFF（将高 16 位全部抹去），读状态等于 S>>>16（无符号补 0 右移 16 位）。当写状态增加1时，等于S+1，当读状态增加1时，等于S+(1<<16)，也就是S+0x00010000。根据状态的划分能得出一个推论：S 不等于 0 时，当写状态（S&0x0000FFFF）等于 0 时，则读状态（S>>>16）大于 0，即读锁已被获取。

2. 写锁的获取与释放
写锁是一个支持重进入的排它锁。如果当前线程已经获取了写锁，则增加写状态。如果当前线程在获取写锁时，读锁已经被获取（读状态不为 0）或者该线程不是已经获取写锁的线程，则当前线程进入等待状态。该方法除了重入条件（当前线程为获取了写锁的线程）之外，增加了一个读锁是否存在的判断。如果存在读锁，则写锁不能被获取，原因在于：读写锁要确保写锁的操作对读锁可见，如果允许读锁在已被获取的情况下对写锁的获取，那么正在运行的其他读线程就无法感知到当前写线程的操作。因此，只有等待其他读线程都释放了读锁，写锁才能被当前线程获取，而写锁一旦被获取，则其他读写线程的后续访问均被阻塞。写锁的释放与 ReentrantLock 的释放过程基本类似，每次释放均减少写状态，当写状态为 0 时表示写锁已被释放，从而等待的读写线程能够继续访问读写锁，同时前次写线程的修改对后续读写线程可见。

3. 读锁的获取与释放
读锁是一个支持重进入的共享锁，它能够被多个线程同时获取，在没有其他写线程访问（或者写状态为 0）时，读锁总会被成功地获取，而所做的也只是（线程安全的）增加读状态。如果当前线程已经获取了读锁，则增加读状态。如果当前线程在获取读锁时，写锁已被其他线程获取，则进入等待状态。读状态是所有线程获取读锁次数的总和，而每个线程各自获取读锁的次数只能选择保存在 ThreadLocal 中，由线程自身维护。在 tryAcquireShared(int unused)方法中，如果其他线程已经获取了写锁，则当前线程获取读锁失败，进入等待状态。如果当前线程获取了写锁或者写锁未被获取，则当前线程（线程安全，依靠 CAS 保证）增加读状态，成功获取读锁。读锁的每次释放（线程安全的，可能有多个读线程同时释放读锁）均减少读状态。
