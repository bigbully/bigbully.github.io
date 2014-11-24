---
layout: post
title: 新版FutureTask
description: 
category: concurrency
---

FutureTask在jdk1.7的时候重写过一次，早先的FutureTask是用AQS实现的，但考虑到在FutureTask取消竞争时仍然保留中断状态这点，新版的FutureTask不再使用AQS，而是直接用CAS来实现。

老板的FutureTask在[AQS](http://bigbully.github.io/AbstractQueuedSynchronizer/)
系列中我做了详细的解析。

虽然没用AQS，不过AQS必备的state可是少不了：

	private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;

FutureTask共包括以上七种状态，由此而生了以下4种状态变迁：

 1. NEW -> COMPLETING -> NORMAL
 2. NEW -> COMPLETING -> EXCEPTIONAL
 3. NEW -> CANCELLED
 4. NEW -> INTERRUPTING -> INTERRUPTED

看来如果我能搞明白这四种状态变迁分别是如何做到的，那么我有理由说我对新版的FutureTask有了完整的认识。

1.正常退出
-------------

当FutureTask被构造时，state状态会首先被设置成NEW

	public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;
    }


这之后会由某一个线程来执行FutureTask的run方法：

	public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }

run方法首先会判断当前状态是否为NEW，然后通过cas设置runner也是运行线程为当前线程。

由于设置state的动作在cas设置当前线程之前，所以为了防止被其他线程改变state的状态值，之后需要double check。

state为NEW确认无误后执行Callable任务并获得返回值。这里通过ran这个变量来记录当前任务是否成功执行。

接下来在set方法中会用cas设置把state变更为COMPLETING，设置返回结果，变更状态为NORMAL，最后执行finishCompletion方法。这里的cas之所以没包括在自旋中，主要是因为只有执行线程才有可能调用，不存在线程争抢的可能。

	private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();

        callable = null;        // to reduce footprint
    }

在finishCompletion外层的使用for循环进行自旋，然后通过cas把保存在等待队列队首的元素置空，接下来把等待队列中所有的线程唤醒。所有这些工作完毕后会执行用于扩展的done方法。

对于其他执行get方法的线程是如何加入到等待队列中的呢？

	public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }


1.异常退出
-------------
    
异常退出与正常退出唯一的区别在于不执行set方法设置返回值，而是执行setException方法设置异常信息，除此之外没有任何区别。

	protected void setException(Throwable t) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = t;
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            finishCompletion();
        }
    }

