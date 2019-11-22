# Table of Contents

  * [使用示例](#使用示例)
  * [ReentrantReadWriteLock 总览](#reentrantreadwritelock-总览)
  * [源码分析](#源码分析)
    * [读锁获取](#读锁获取)
    * [读锁释放](#读锁释放)
    * [写锁获取](#写锁获取)
    * [写锁释放](#写锁释放)
  * [锁降级](#锁降级)
  * [总结](#总结)


本文转自：https://www.javadoop.com/

**本文转载自互联网，侵删**

本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

文章同步发于我的个人博客：
> www.how2playlife.com

本文是微信公众号【Java技术江湖】的《Java并发指南》其中一篇，本文大部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何全面深入地学习Java并发技术，从Java多线程基础，再到并发编程的基础知识，从Java并发包的入门和实战，再到JUC的源码剖析，一步步地学习Java并发编程，并上手进行实战，以便让你更完整地了解整个Java并发编程知识体系，形成自己的知识框架。

为了更好地总结和检验你的学习成果，本系列文章也会提供一些对应的面试题以及参考答案。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。
<!--more -->

本文内容：读写锁 ReentrantReadWriteLock 的源码分析，基于 Java7/Java8。

阅读建议：虽然我这里会介绍一些 AQS 的知识，不过如果你完全不了解 AQS，看本文就有点吃力了。

## 使用示例

下面这个例子非常实用，我是 javadoc 的搬运工：

```
// 这是一个关于缓存操作的故事
class CachedData {
    Object data;
    volatile boolean cacheValid;
    // 读写锁实例
    final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();

    void processCachedData() {
        // 获取读锁
        rwl.readLock().lock();
        if (!cacheValid) { // 如果缓存过期了，或者为 null
            // 释放掉读锁，然后获取写锁 (后面会看到，没释放掉读锁就获取写锁，会发生死锁情况)
            rwl.readLock().unlock();
            rwl.writeLock().lock();

            try {
                if (!cacheValid) { // 重新判断，因为在等待写锁的过程中，可能前面有其他写线程执行过了
                    data = ...
                    cacheValid = true;
                }
                // 获取读锁 (持有写锁的情况下，是允许获取读锁的，称为 “锁降级”，反之不行。)
                rwl.readLock().lock();
            } finally {
                // 释放写锁，此时还剩一个读锁
                rwl.writeLock().unlock(); // Unlock write, still hold read
            }
        }

        try {
            use(data);
        } finally {
            // 释放读锁
            rwl.readLock().unlock();
        }
    }
}
```

ReentrantReadWriteLock 分为读锁和写锁两个实例，读锁是共享锁，可被多个线程同时使用，写锁是独占锁。持有写锁的线程可以继续获取读锁，反之不行。

## ReentrantReadWriteLock 总览

这一节比较重要，我们要先看清楚 ReentrantReadWriteLock 的大框架，然后再到源码细节。

首先，我们来看下 ReentrantReadWriteLock 的结构，它有好些嵌套类：

![11](https://www.javadoop.com/blogimages/reentrant-read-write-lock/11.png)

大家先仔细看看这张图中的信息。然后我们把 ReadLock 和 WriteLock 的代码提出来一起看，清晰一些：

![12](https://www.javadoop.com/blogimages/reentrant-read-write-lock/12.png)

很清楚了，ReadLock 和 WriteLock 中的方法都是通过 Sync 这个类来实现的。Sync 是 AQS 的子类，然后再派生了公平模式和不公平模式。

从它们调用的 Sync 方法，我们可以看到： **ReadLock 使用了共享模式，WriteLock 使用了独占模式**。

等等，**同一个 AQS 实例怎么可以同时使用共享模式和独占模式**？？？

这里给大家回顾下 AQS，我们横向对比下 AQS 的共享模式和独占模式：

![13](https://www.javadoop.com/blogimages/reentrant-read-write-lock/13.png)

AQS 的精髓在于内部的属性 **state**：

1.  对于独占模式来说，通常就是 0 代表可获取锁，1 代表锁被别人获取了，重入例外
2.  而共享模式下，每个线程都可以对 state 进行加减操作

也就是说，独占模式和共享模式对于 state 的操作完全不一样，那读写锁 ReentrantReadWriteLock 中是怎么使用 state 的呢？答案是**将 state 这个 32 位的 int 值分为高 16 位和低 16位，分别用于共享模式和独占模式**。

## 源码分析

有了前面的概念，大家心里应该都有数了吧，下面就不再那么啰嗦了，直接代码分析。

源代码加注释 1500 行，并不算难，我们要看的代码量不大。如果你前面一节都理解了，那么直接从头开始一行一行往下看就是了，还是比较简单的。

ReentrantReadWriteLock 的前面几行很简单，我们往下滑到 Sync 类，先来看下它的所有的属性：

```
abstract static class Sync extends AbstractQueuedSynchronizer {
    // 下面这块说的就是将 state 一分为二，高 16 位用于共享模式，低16位用于独占模式
    static final int SHARED_SHIFT   = 16;
    static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
    static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
    static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;
    // 取 c 的高 16 位值，代表读锁的获取次数(包括重入)
    static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
    // 取 c 的低 16 位值，代表写锁的重入次数，因为写锁是独占模式
    static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }

    // 这个嵌套类的实例用来记录每个线程持有的读锁数量(读锁重入)
    static final class HoldCounter {
        // 持有的读锁数
        int count = 0;
        // 线程 id
        final long tid = getThreadId(Thread.currentThread());
    }

    // ThreadLocal 的子类
    static final class ThreadLocalHoldCounter
        extends ThreadLocal<HoldCounter> {
        public HoldCounter initialValue() {
            return new HoldCounter();
        }
    }
    /**
      * 组合使用上面两个类，用一个 ThreadLocal 来记录当前线程持有的读锁数量
      */ 
    private transient ThreadLocalHoldCounter readHolds;

    // 用于缓存，记录"最后一个获取读锁的线程"的读锁重入次数，
    // 所以不管哪个线程获取到读锁后，就把这个值占为已用，这样就不用到 ThreadLocal 中查询 map 了
    // 算不上理论的依据：通常读锁的获取很快就会伴随着释放，
    //   显然，在 获取->释放 读锁这段时间，如果没有其他线程获取读锁的话，此缓存就能帮助提高性能
    private transient HoldCounter cachedHoldCounter;

    // 第一个获取读锁的线程(并且其未释放读锁)，以及它持有的读锁数量
    private transient Thread firstReader = null;
    private transient int firstReaderHoldCount;

    Sync() {
        // 初始化 readHolds 这个 ThreadLocal 属性
        readHolds = new ThreadLocalHoldCounter();
        // 为了保证 readHolds 的内存可见性
        setState(getState()); // ensures visibility of readHolds
    }
    ...
}
```

1.  state 的高 16 位代表读锁的获取次数，包括重入次数，获取到读锁一次加 1，释放掉读锁一次减 1
2.  state 的低 16 位代表写锁的获取次数，因为写锁是独占锁，同时只能被一个线程获得，所以它代表重入次数
3.  每个线程都需要维护自己的 HoldCounter，记录该线程获取的读锁次数，这样才能知道到底是不是读锁重入，用 ThreadLocal 属性 **readHolds** 维护
4.  **cachedHoldCounter** 有什么用？其实没什么用，但能提示性能。将最后一次获取读锁的线程的 HoldCounter 缓存到这里，这样比使用 ThreadLocal 性能要好一些，因为 ThreadLocal 内部是基于 map 来查询的。但是 cachedHoldCounter 这一个属性毕竟只能缓存一个线程，所以它要起提升性能作用的依据就是：通常读锁的获取紧随着就是该读锁的释放。我这里可能表达不太好，但是大家应该是懂的吧。
5.  **firstReader** 和 **firstReaderHoldCount** 有什么用？其实也没什么用，但是它也能提示性能。将"第一个"获取读锁的线程记录在 firstReader 属性中，这里的**第一个**不是全局的概念，等这个 firstReader 当前代表的线程释放掉读锁以后，会有后来的线程占用这个属性的。**firstReader 和 firstReaderHoldCount 使得在读锁不产生竞争的情况下，记录读锁重入次数非常方便快速**
6.  如果一个线程使用了 firstReader，那么它就不需要占用 cachedHoldCounter
7.  个人认为，读写锁源码中最让初学者头疼的就是这几个用于提升性能的属性了，使得大家看得云里雾里的。主要是因为 ThreadLocal 内部是通过一个 ThreadLocalMap 来操作的，会增加检索时间。而很多场景下，执行 unlock 的线程往往就是刚刚最后一次执行 lock 的线程，中间可能没有其他线程进行 lock。还有就是很多不怎么会发生读锁竞争的场景。

上面说了这么多，是希望能帮大家降低后面阅读源码的压力，大家也可以先看看后面的，然后再慢慢体会。

前面我们好像都只说读锁，完全没提到写锁，主要是因为写锁真的是简单很多，我也特地将写锁的源码放到了后面，我们先啃下最难的读锁先。

### 读锁获取

下面我就不一行一行按源码顺序说了，我们按照使用来说。

我们来看下读锁 ReadLock 的 lock 流程：

```
// ReadLock
public void lock() {
    sync.acquireShared(1);
}
// AQS
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

然后我们就会进到 Sync 类的 tryAcquireShared 方法：

> 在 AQS 中，如果 tryAcquireShared(arg) 方法返回值小于 0 代表没有获取到共享锁(读锁)，大于 0 代表获取到
> 
> 回顾 AQS 共享模式：tryAcquireShared 方法不仅仅在 acquireShared 的最开始被使用，这里是 try，也就可能会失败，如果失败的话，执行后面的 doAcquireShared，进入到阻塞队列，然后等待前驱节点唤醒。唤醒以后，还是会调用 tryAcquireShared 进行获取共享锁的。当然，唤醒以后再 try 是很容易获得锁的，因为这个节点已经排了很久的队了，组织是会照顾它的。
> 
> 所以，你在看下面这段代码的时候，要想象到两种获取读锁的场景，一种是新来的，一种是排队排到它的。

```
protected final int tryAcquireShared(int unused) {

    Thread current = Thread.currentThread();
    int c = getState();

    // exclusiveCount(c) 不等于 0，说明有线程持有写锁，
    //    而且不是当前线程持有写锁，那么当前线程获取读锁失败
    //         （另，如果持有写锁的是当前线程，是可以继续获取读锁的）
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;

    // 读锁的获取次数
    int r = sharedCount(c);

    // 读锁获取是否需要被阻塞，稍后细说。为了进去下面的分支，假设这里不阻塞就好了
    if (!readerShouldBlock() &&
        // 判断是否会溢出 (2^16-1，没那么容易溢出的)
        r < MAX_COUNT &&
        // 下面这行 CAS 是将 state 属性的高 16 位加 1，低 16 位不变，如果成功就代表获取到了读锁
        compareAndSetState(c, c + SHARED_UNIT)) {

        // =======================
        //   进到这里就是获取到了读锁
        // =======================

        if (r == 0) {
            // r == 0 说明此线程是第一个获取读锁的，或者说在它前面获取读锁的都走光光了，它也算是第一个吧
            //  记录 firstReader 为当前线程，及其持有的读锁数量：1
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            // 进来这里，说明是 firstReader 重入获取读锁（这非常简单，count 加 1 结束）
            firstReaderHoldCount++;
        } else {
            // 前面我们说了 cachedHoldCounter 用于缓存最后一个获取读锁的线程
            // 如果 cachedHoldCounter 缓存的不是当前线程，设置为缓存当前线程的 HoldCounter
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0) 
                // 到这里，那么就是 cachedHoldCounter 缓存的是当前线程，但是 count 为 0，
                // 大家可以思考一下：这里为什么要 set ThreadLocal 呢？(当然，答案肯定不在这块代码中)
                //   既然 cachedHoldCounter 缓存的是当前线程，
                //   当前线程肯定调用过 readHolds.get() 进行初始化 ThreadLocal
                readHolds.set(rh);

            // count 加 1
            rh.count++;
        }
        // return 大于 0 的数，代表获取到了共享锁
        return 1;
    }
    // 往下看
    return fullTryAcquireShared(current);
}
```

上面的代码中，要进入 if 分支，需要满足：readerShouldBlock() 返回 false，并且 CAS 要成功（我们先不要纠结 MAX_COUNT 溢出）。

那我们反向推，怎么样进入到最后的 fullTryAcquireShared：

*   readerShouldBlock() 返回 true，2 种情况：

    *   在 FairSync 中说的是 hasQueuedPredecessors()，即阻塞队列中有其他元素在等待锁。

        > 也就是说，公平模式下，有人在排队呢，你新来的不能直接获取锁

    *   在 NonFairSync 中说的是 apparentlyFirstQueuedIsExclusive()，即判断阻塞队列中 head 的第一个后继节点是否是来获取写锁的，如果是的话，让这个写锁先来，避免写锁饥饿。

        > 作者给写锁定义了更高的优先级，所以如果碰上获取写锁的线程**马上**就要获取到锁了，获取读锁的线程不应该和它抢。
        > 
        > 如果 head.next 不是来获取写锁的，那么可以随便抢，因为是非公平模式，大家比比 CAS 速度

*   compareAndSetState(c, c + SHARED_UNIT) 这里 CAS 失败，存在竞争。可能是和另一个读锁获取竞争，当然也可能是和另一个写锁获取操作竞争。

然后就会来到 fullTryAcquireShared 中再次尝试：

```
/**
 * 1\. 刚刚我们说了可能是因为 CAS 失败，如果就此返回，那么就要进入到阻塞队列了，
 *    想想有点不甘心，因为都已经满足了 !readerShouldBlock()，也就是说本来可以不用到阻塞队列的，
 *    所以进到这个方法其实是增加 CAS 成功的机会
 * 2\. 在 NonFairSync 情况下，虽然 head.next 是获取写锁的，我知道它等待很久了，我没想和它抢，
 *    可是如果我是来重入读锁的，那么只能表示对不起了
 */
final int fullTryAcquireShared(Thread current) {
    HoldCounter rh = null;
    // 别忘了这外层有个 for 循环
    for (;;) {
        int c = getState();
        // 如果其他线程持有了写锁，自然这次是获取不到读锁了，乖乖到阻塞队列排队吧
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1;
            // else we hold the exclusive lock; blocking here
            // would cause deadlock.
        } else if (readerShouldBlock()) {
            /**
              * 进来这里，说明：
              *  1\. exclusiveCount(c) == 0：写锁没有被占用
              *  2\. readerShouldBlock() 为 true，说明阻塞队列中有其他线程在等待
              *
              * 既然 should block，那进来这里是干什么的呢？
              * 答案：是进来处理读锁重入的！
              * 
              */

            // firstReader 线程重入读锁，直接到下面的 CAS
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
            } else {
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        // cachedHoldCounter 缓存的不是当前线程
                        // 那么到 ThreadLocal 中获取当前线程的 HoldCounter
                        // 如果当前线程从来没有初始化过 ThreadLocal 中的值，get() 会执行初始化
                        rh = readHolds.get();
                        // 如果发现 count == 0，也就是说，纯属上一行代码初始化的，那么执行 remove
                        // 然后往下两三行，乖乖排队去
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                if (rh.count == 0)
                    // 排队去。
                    return -1;
            }
            /**
              * 这块代码我看了蛮久才把握好它是干嘛的，原来只需要知道，它是处理重入的就可以了。
              * 就是为了确保读锁重入操作能成功，而不是被塞到阻塞队列中等待
              *
              * 另一个信息就是，这里对于 ThreadLocal 变量 readHolds 的处理：
              *    如果 get() 后发现 count == 0，居然会做 remove() 操作，
              *    这行代码对于理解其他代码是有帮助的
              */
        }

        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");

        if (compareAndSetState(c, c + SHARED_UNIT)) {
            // 这里 CAS 成功，那么就意味着成功获取读锁了
            // 下面需要做的是设置 firstReader 或 cachedHoldCounter

            if (sharedCount(c) == 0) {
                // 如果发现 sharedCount(c) 等于 0，就将当前线程设置为 firstReader
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                // 下面这几行，就是将 cachedHoldCounter 设置为当前线程
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh;
            }
            // 返回大于 0 的数，代表获取到了读锁
            return 1;
        }
    }
}
```

> firstReader 是每次将**读锁获取次数**从 0 变为 1 的那个线程。
> 
> 能缓存到 firstReader 中就不要缓存到 cachedHoldCounter 中。

上面的源码分析应该说得非常详细了，如果到这里你不太能看懂上面的有些地方的注释，那么可以先往后看，然后再多看几遍。

### 读锁释放

下面我们看看读锁释放的流程：

```
// ReadLock
public void unlock() {
    sync.releaseShared(1);
}
```

```
// Sync
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared(); // 这句代码其实唤醒 获取写锁的线程，往下看就知道了
        return true;
    }
    return false;
}

// Sync
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    if (firstReader == current) {
        if (firstReaderHoldCount == 1)
            // 如果等于 1，那么这次解锁后就不再持有锁了，把 firstReader 置为 null，给后来的线程用
            // 为什么不顺便设置 firstReaderHoldCount = 0？因为没必要，其他线程使用的时候自己会设值
            firstReader = null;
        else
            firstReaderHoldCount--;
    } else {
        // 判断 cachedHoldCounter 是否缓存的是当前线程，不是的话要到 ThreadLocal 中取
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current))
            rh = readHolds.get();

        int count = rh.count;
        if (count <= 1) {

            // 这一步将 ThreadLocal remove 掉，防止内存泄漏。因为已经不再持有读锁了
            readHolds.remove();

            if (count <= 0)
                // 就是那种，lock() 一次，unlock() 好几次的逗比
                throw unmatchedUnlockException();
        }
        // count 减 1
        --rh.count;
    }

    for (;;) {
        int c = getState();
        // nextc 是 state 高 16 位减 1 后的值
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            // 如果 nextc == 0，那就是 state 全部 32 位都为 0，也就是读锁和写锁都空了
            // 此时这里返回 true 的话，其实是帮助唤醒后继节点中的获取写锁的线程
            return nextc == 0;
    }
}
```

读锁释放的过程还是比较简单的，主要就是将 hold count 减 1，如果减到 0 的话，还要将 ThreadLocal 中的 remove 掉。

然后是在 for 循环中将 state 的高 16 位减 1，如果发现读锁和写锁都释放光了，那么唤醒后继的获取写锁的线程。

### 写锁获取

1.  写锁是独占锁。
2.  如果有读锁被占用，写锁获取是要进入到阻塞队列中等待的。

```
// WriteLock
public void lock() {
    sync.acquire(1);
}
// AQS
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        // 如果 tryAcquire 失败，那么进入到阻塞队列等待
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

// Sync
protected final boolean tryAcquire(int acquires) {

    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {

        // 看下这里返回 false 的情况：
        //   c != 0 && w == 0: 写锁可用，但是有线程持有读锁(也可能是自己持有)
        //   c != 0 && w !=0 && current != getExclusiveOwnerThread(): 其他线程持有写锁
        //   也就是说，只要有读锁或写锁被占用，这次就不能获取到写锁
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;

        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");

        // 这里不需要 CAS，仔细看就知道了，能到这里的，只可能是写锁重入，不然在上面的 if 就拦截了
        setState(c + acquires);
        return true;
    }

    // 如果写锁获取不需要 block，那么进行 CAS，成功就代表获取到了写锁
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

下面看一眼 **writerShouldBlock()** 的判定，然后你再回去看一篇写锁获取过程。

```
static final class NonfairSync extends Sync {
    // 如果是非公平模式，那么 lock 的时候就可以直接用 CAS 去抢锁，抢不到再排队
    final boolean writerShouldBlock() {
        return false; // writers can always barge
    }
    ...
}
static final class FairSync extends Sync {
    final boolean writerShouldBlock() {
        // 如果是公平模式，那么如果阻塞队列有线程等待的话，就乖乖去排队
        return hasQueuedPredecessors();
    }
    ...
}
```

### 写锁释放

```
// WriteLock
public void unlock() {
    sync.release(1);
}

// AQS
public final boolean release(int arg) {
    // 1\. 释放锁
    if (tryRelease(arg)) {
        // 2\. 如果独占锁释放"完全"，唤醒后继节点
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

// Sync 
// 释放锁，是线程安全的，因为写锁是独占锁，具有排他性
// 实现很简单，state 减 1 就是了
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    setState(nextc);
    // 如果 exclusiveCount(nextc) == 0，也就是说包括重入的，所有的写锁都释放了，
    // 那么返回 true，这样会进行唤醒后继节点的操作。
    return free;
}
```

看到这里，是不是发现写锁相对于读锁来说要简单很多。

## 锁降级

Doug Lea 没有说写锁更**高级**，如果有线程持有读锁，那么写锁获取也需要等待。

不过从源码中也可以看出，确实会给写锁一些特殊照顾，如非公平模式下，为了提高吞吐量，lock 的时候会先 CAS 竞争一下，能成功就代表读锁获取成功了，但是如果发现 head.next 是获取写锁的线程，就不会去做 CAS 操作。

Doug Lea 将持有写锁的线程，去获取读锁的过程称为**锁降级（Lock downgrading）**。这样，此线程就既持有写锁又持有读锁。

但是，**锁升级**是不可以的。线程持有读锁的话，在没释放的情况下不能去获取写锁，因为会发生**死锁**。

回去看下写锁获取的源码：

```
protected final boolean tryAcquire(int acquires) {

    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        // 看下这里返回 false 的情况：
        //   c != 0 && w == 0: 写锁可用，但是有线程持有读锁(也可能是自己持有)
        //   c != 0 && w !=0 && current != getExclusiveOwnerThread(): 其他线程持有写锁
        //   也就是说，只要有读锁或写锁被占用，这次就不能获取到写锁
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        ...
    }
    ...
}
```

仔细想想，如果线程 a 先获取了读锁，然后获取写锁，那么线程 a 就到阻塞队列休眠了，自己把自己弄休眠了，而且可能之后就没人去唤醒它了。

## 总结

![14](https://www.javadoop.com/blogimages/reentrant-read-write-lock/14.png)

（全文完）
