---
layout: post
title: AbstractQueuedSynchronizer
description: 并发的灵魂
category: concurrency
---


如果不读源码，我不会知道AbstractQueuedSynchronizer，更不会认识到它使整个java.util.concurrent包中众多并发工具类的灵魂。AbstractQueuedSynchronizer官方的推荐用法是，在并发工具类内部使用一个同步器sync实现来继承AbstractQueuedSynchronizer，并把预留的抽象方法(例如acquire和release)赋予业务含义后，同步相关的操作就交由sync来执行了。

查看源码可以看到,AbstractQueuedSynchronizer提供了一个基于FIFO队列,FIFO队列使用的是链表的数据结构,其中的每一个node都拥有自身的可改变的状态waitStatus,而node自身又可以获得他的前驱node和后继node,每个node封装了当前所在线程.AbstractQueuedSynchronizer会通过响应不同的方法,根据node不同的waitStatus,达到操作线程的目的.

我认为首先需要明确的是node的waitStatus：

>**waitStatus:**

> - 1.CANCELLED. 表示当前节点由于超时或终端而需要被取消
> - -1.SIGNAL.表示当前节点的后继节点包含的线程需要被唤醒
> - -2.CONDITION.表示当前节点在等待condition，也就是在condition队列中
> - -3.PROPAGATE.只有在队列头会被设置，表示releaseShared需要被传播给后续节点。
> - 0.不属于上述的任何一种。用来表示正常处于同步状态的node，初始化时会被设置为0.

用数字标示纯粹是为了运算方便。负数用来表示当前节点不需要被唤醒。

每个node还有一个类型属性：共享（shared）或独占（exclusive）。通常不会同时出现在FIFO队列中，不过ReadWriteLock是一个列外。共享模式指的是允许多个线程获取同一个锁而且可能获取成功，独占模式指的是一个锁如果被一个线程持有，其他线程必须等待。多个线程读取一个文件可以采用共享模式，而当有一个线程在写文件时不会允许另一个线程写这个文件，这就是独占模式的应用场景。

整理完这些基础知识之后，在这里先拿闭锁CountDownLatch举个例子，因为闭锁相对来说最常见，用法也最单一。因此它的实现也比较简单。（我非常希望能在这篇笔记中把所有涉及到AbstractQueuedSynchronizer的并发工具都研究一遍。）


CountDownLatch
---------------------

闭锁CountDownLatch的用法实在太简单了。在这里给个小例子：

	class Driver { // ...

        void main() throws InterruptedException {
            CountDownLatch startSignal = new CountDownLatch(1);
            CountDownLatch doneSignal = new CountDownLatch(N);

            for (int i = 0; i < N; ++i) // create and start threads
                new Thread(new Worker(startSignal, doneSignal)).start();

            doSomethingElse();            // don't let run yet
            startSignal.countDown();      // let all threads proceed
            doSomethingElse();
            doneSignal.await();           // wait for all to finish
        }
    }

    class Worker implements Runnable {
        private final CountDownLatch startSignal;
        private final CountDownLatch doneSignal;

        Worker(CountDownLatch startSignal, CountDownLatch doneSignal) {
            this.startSignal = startSignal;
            this.doneSignal = doneSignal;
        }

        public void run() {
            try {
                startSignal.await();
                doWork();
                doneSignal.countDown();
            } catch (InterruptedException ex) {
            } // return;
        }

        void doWork() {...}
    }

之后来看看CountDownLatch内部都做了哪些工作。唯一的构造方法是传入一个count值，初始化count为多少，就代表这个闭锁可以countDown多少次。

	public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }

在构造函数中创建了一个同步器Sync。这个sync继承了AbstractQueuedSynchronizer，并完成了CountDownLatch的所有工作。

	private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            return getState() == 0? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
	
从源码中可以看到由CountDownLatch的构造函数传入的count被设置为sync的state，state在AbstractQueuedSynchronizer内部是volatile修饰的，它的状态改变对所有线程可见。

AbstractQueuedSynchronizer会强制他的子类实现tryAcquire(或tryAcquireShared)，tryRelease(或tryReleaseShared)方法，从名称上可以看出，分别代表独占式和共享式.在这里CountDownLatch的Sync是共享式的同步器。

先看一下CountDownLatch的await方法是如何实现的。当某个线程CountDownLatch调用await方法时，会在这里阻塞，等待CountDownLatch调用countDown方法的次数超过count值，才被唤醒。
	
	public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

		public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }

acquireSharedInterruptibly方法是在AbstractQueuedSynchronizer已经实现好的。首先他会检查当前线程是否已经被中断，然后调用用户自行实现的tryAcquireShared方法检查是否满足状态，tryAcquireShared方法的返回值为int类型。方法规定:

>**返回值**

> - 大于0.表示本次尝试获取锁成功，并且后续的其他线程再次尝试获取锁仍然有可能成功(后续的其他线程需要去检查是否能获取锁)
> - 等于0.表示本次尝试成功，但后续的其他线程不会成功获取锁
> - 小于0.表示本次尝试失败.

再来看一下CountDownLatch实现的tryAcquireShared方法体:

	public int tryAcquireShared(int acquires) {
            return getState() == 0? 1 : -1;
    }

可以看到只有getState()，也就是构造函数中的count值为0时，才会返回1，即能够获取锁，并且后续的其他线程再次尝试获取锁仍然有可能成功。反之，则失败。

当CountDownLatch从未countDown过，自然在await时会失败，继而调用doAcquireSharedInterruptibly方法。

	private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {              
                        setHeadAndPropagate(node, r);//把自己设置为head节点并唤醒后继节点
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

认真分析一下这个方法.

 1. 首先创建共享类型的node并加入到FIFO队列中
 2. 获取node的前驱节点p
 3. 如果前驱节点p为head节点，即当前节点很有可能能获得锁，则再次尝试获取锁tryAcquireShared
 4. 如果获取锁成功,则执行setHeadAndPropagate,用来把当前node设置为head结点,并向后传播自己获取锁成功的信息.
 5. 如果前驱节点p不是head节点，或者p虽然是head节点，但当前节点没有成功获得锁。则检查是否需要让线程等待（park），如果需要，则进入等待状态。
 6. 在当前线程被唤醒后，查看是否是因为中断而被唤醒的。如果是因为中断被唤醒的，直接中断，并在finally中取消这次获取锁的操作（cancelAcquire），即从队列中删除当前节点，并顺便从队列中剔除已经cancel的节点，如果需要，唤醒当前节点的后继节点。

再来看看addWaiter方法：

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

上面两个方法可以合在一起看

 1. addWaiter时首先迅速判断一下队列是不是已经建立好了，即tail节点是不是已经存在了，如果存在，则把当前节点加到队列末尾。
 2. 如果不存在，则进入enq方法，double check一下，确定队列仍然没有创建，则通过cas的原子方法创建队列。

当有两个线程都先后尝试获取锁的时候，FIFO队列的快照和node状态如图所示：

![image](http://bigbully.github.io/images/aqs-acquireShare.png)

可以看到第一个线程T1尝试获取锁没成功的时候，会初始化这个队列，创建特殊节点，head node。并把head node的状态从SYNC切换到SIGNAL，自身维持SYNC的状态。当第二个线程T2，尝试获取锁失败时，会把他的前驱节点T1的状态从SYNC切换到SIGNAL，自身维持SYNC的状态。

我还是觉得非常有必要再看一眼shouldParkAfterFailedAcquire方法，这个方法也是我最疑惑的一点。

	private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

因为我很疑惑，所以在这里写下的东西只是我的理解。这个方法总共有几个判断条件。

 1. 如果前驱节点的状态是SIGNAL，则返回true，也就是让当前节点立刻休眠。
 2. 如果前驱结点被取消，则尝试在队列中向前查找，找到一个没有被取消的节点，与当前节点关联上。这意味着在这里也会从队列中删除掉所有的CANCEL状态的节点。这个动作实际上发生在很多地方，因为设置节点的CANCEL状态和删除CANCEL并不是串行的。
 3. 如果前驱结点状态为SYNC或PROPAGATE，则设置它的状态为SIGNAL，并返回false，即当前节点先不休眠，尝试自旋一次后再次进行获取锁的操作。

到此为止CountDownLatch.await方法就都解析完了。再来看看当执行CountDownLatch.countDown时,同步器都做了些什么。

	public void countDown() {
        sync.releaseShared(1);
    }

	public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }

可以看到countDown方法实际上执行的是AbstractQueuedSynchronizer中的releaseShared方法，从方法名上得知也是共享模式下专用的方法。首先会去验证一下能否释放锁，tryReleaseShared是在CountDownLatch的同步器中实现的，在这里可以再回顾一下。用自旋和cas保证设置把count值--的原子性，当count=0时直接返回false。

	protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
               return nextc == 0;
        }
    }

尝试获得锁成功之后需要执行doReleaseShared做之后的处理。

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

