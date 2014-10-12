---
layout: post
title: 延迟队列
description: DelayQueue学习笔记
category: concurrency
---

今天打算研究一下java并发包中的延迟队列DelayQueue，不过这之前有必要好好看一下优先级队列PriorityQueue是如何实现的，因为DelayQueue实际上是依托PriorityQueue来实现的。

首先，PriorityQueue的几个构造函数需要注意一下：

	public PriorityQueue(int initialCapacity,
                         Comparator<? super E> comparator) {
        // Note: This restriction of at least one is not actually needed,
        // but continues for 1.5 compatibility
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.queue = new Object[initialCapacity];
        this.comparator = comparator;
    }
    
    @SuppressWarnings("unchecked")
    public PriorityQueue(Collection<? extends E> c) {
        if (c instanceof SortedSet<?>) {
            SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
            this.comparator = (Comparator<? super E>) ss.comparator();
            initElementsFromCollection(ss);
        }
        else if (c instanceof PriorityQueue<?>) {
            PriorityQueue<? extends E> pq = (PriorityQueue<? extends E>) c;
            this.comparator = (Comparator<? super E>) pq.comparator();
            initFromPriorityQueue(pq);
        }
        else {
            this.comparator = null;
            initFromCollection(c);
        }
    }
    
PriorityQueue中的没有个元素都是有大小之分的，index=0的元素是整个队列中最小元素，所以第一个构造函数中除了需要传入常规的队列初始大小的参数外，也可以选择传入比较器Comparator。如果初始化时传入了Comparator，则每一个元素会使用Comparator比较大小，如果没有传入Comparator，则队列中的元素必须实现了Comparable接口。

第二个构造函数是常见的把某个Collection转化成PriorityQueue，因为Collection中有SortedSet和PriorityQueue自带Comparator，所以会进行特殊处理。

我们知道PriorityQueue是使用二叉堆来构建的，而二叉堆是可以用一维数组来表示的，所以构建二叉堆的过程就是首先把collection中的元素copy到数组中，再对所有元素依次执行下滤操作运行时间为O(N).

	private void initFromCollection(Collection<? extends E> c) {
        initElementsFromCollection(c);
        heapify();
    }
    
	private void heapify() {
        for (int i = (size >>> 1) - 1; i >= 0; i--)
            siftDown(i, (E) queue[i]);
    }
   
往PriorityQueue中插入一个元素的方法其实很简单，首先通过modCount进行并发控制，之后查看是否需要扩容，最后进行上滤操作。完成。

	public boolean offer(E e) {
        if (e == null)
            throw new NullPointerException();
        modCount++;
        int i = size;
        if (i >= queue.length)
            grow(i + 1);
        size = i + 1;
        if (i == 0)
            queue[0] = e;
        else
            siftUp(i, e);
        return true;
    }
    
根据二叉堆的性质，PriorityQueue中最小的元素就是数组中的第一个元素。所以peek操作很简单。关键是poll操作。poll操作不但要返回数组中的第一个元素，同时还要保持堆序性，所以要把最后一个元素放置到堆顶，然后执行下滤操作，保证堆顶的元素最小。

	public E peek() {
        if (size == 0)
            return null;
        return (E) queue[0];
    }
    
    public E poll() {
        if (size == 0)
            return null;
        int s = --size;
        modCount++;
        E result = (E) queue[0];
        E x = (E) queue[s];
        queue[s] = null;
        if (s != 0)
            siftDown(0, x);
        return result;
    }
    
此外，PriorityQueue还提供了移除队列中任何一个元素的方法，这个方法比poll方法只复杂一步，因为不但要移除这个元素，还要保持堆序性，而且被移除的元素不像poll方法中移除的第一个元素一样可以保证是最小元素，所以在removeAt方法中，要尝试进行下滤和上滤操作，把末尾的元素放到合适的位置中。

	public boolean remove(Object o) {
        int i = indexOf(o);
        if (i == -1)
            return false;
        else {
            removeAt(i);
            return true;
        }
    }
    
    private E removeAt(int i) {
        assert i >= 0 && i < size;
        modCount++;
        int s = --size;
        if (s == i) // removed last element
            queue[i] = null;
        else {
            E moved = (E) queue[s];
            queue[s] = null;
            siftDown(i, moved);
            if (queue[i] == moved) {
                siftUp(i, moved);
                if (queue[i] != moved)
                    return moved;
            }
        }
        return null;
    }
    
值得注意的是任何实现Collection接口的集合都会提供的toArray方法，PriorityQueue也不例外，不过PriorityQueue的toArray方法得到的array只是简单的返回内部数组的copy，不保证任何有序性。

对PriorityQueue的学习就到这里，下面来看看DelayQueue是怎么做的。

看了DelayQueue也算开了眼界了，尤其是线程leader-follower的运用，太巧妙了。kafka中多个线程对queue的生产消费之间的协作也是参照这种模式来做的。

DelayQueue中有几个关键属性，分别是可重入锁lock，优先级队列q，主导线程leader（永远是那个等待第一个元素过期的线程，不过并不一定是返回这个元素的线程），以及用于阻塞的Condition available。生产和消费延迟元素Delayed就依靠一下几个属性来完成的。

	private transient final ReentrantLock lock = new ReentrantLock();
    private final PriorityQueue<E> q = new PriorityQueue<E>();
    private Thread leader = null;
    private final Condition available = lock.newCondition();
    
提到延迟元素Delayed，也就是被放入DelayQueue需要实现的接口。它集成了Comparable接口，而且提供了待实现的方法getDelay，compareTo(T o)方法用来决定延迟元素在PriorityQueue队列中的顺序，getDelay(TimeUnit unit)方法用来计算出当前元素距离过期所剩的时间。

	public interface Delayed extends Comparable<Delayed> {
		long getDelay(TimeUnit unit);
	}
	
首先看一下如何把延迟元素放入队列的操作，为了保证线程安全，所以在offer操作时加锁。在把元素放入PriorityQueue之后还会判断当前放入的元素是否被放置在PriorityQueue的首位，也就是最先过期的元素，如果是的话，会清空leader线程的引用，然后唤醒所有在poll过程中等待的线程。

	public boolean offer(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            q.offer(e);
            if (q.peek() == e) {
                leader = null;
                available.signal();
            }
            return true;
        } finally {
            lock.unlock();
        }
    }
    
接下来看一下poll方法，因为poll方法是非阻塞的，所以实现起来异常简单。找出第一个元素，如果过期了就返回这个元素，如果没过期就返回null。

	public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            E first = q.peek();
            if (first == null || first.getDelay(TimeUnit.NANOSECONDS) > 0)
                return null;
            else
                return q.poll();
        } finally {
            lock.unlock();
        }
    } 
    
DelayQueue中真正复杂的方法时如何处理阻塞方法。take方法的用处在于如果没有任何元素过期，则一直阻塞。

我们看到方法体内有一个自旋，自旋内部首先获取第一个元素，如果没有的话，等待offer操作完成并被唤醒。

如果能够获得的第一个元素，当发现元素已经过期时，则直接返回。

如果没有过期而且自身是follower时（意味着有其他线程是leader线程），则会等待被leader线程唤醒，只有leader线程会等待到第一个元素过期，并尝试返回这个元素。当然这时有其他线程在leader线程等待过程中获得锁，也有可能返回这个过期元素。

如果当时没有leader线程，则自己成为leader线程，等待元素过期。过期之后，自身不再处于leader线程这个角色，进入下一次自旋，尝试把元素返回。如果还没有任何线程抢占成leader线程，并且队列中有元素存在，则唤醒所有等待的follower线程。

	public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                E first = q.peek();
                if (first == null)
                    available.await();
                else {
                    long delay = first.getDelay(TimeUnit.NANOSECONDS);
                    if (delay <= 0)
                        return q.poll();
                    else if (leader != null)
                        available.await();
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            available.awaitNanos(delay);
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }
    
再来看看带超时时间的poll方法时怎么做的。

还是先获得锁，然后在自旋内部检查是否存在第一个元素。如果不存在，而且等待超时，则直接返回，如果确实需要等待，则等待到超时，然后再次自旋，如果还没有第一个元素，则返回null。

后续处理其实和阻塞的take方法差不多，只不过要多考虑时间维度的限制条件。在此不再过多解释。



	public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                E first = q.peek();
                if (first == null) {
                    if (nanos <= 0)
                        return null;
                    else
                        nanos = available.awaitNanos(nanos);
                } else {
                    long delay = first.getDelay(TimeUnit.NANOSECONDS);
                    if (delay <= 0)
                        return q.poll();
                    if (nanos <= 0)
                        return null;
                    if (nanos < delay || leader != null)
                        nanos = available.awaitNanos(nanos);
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            long timeLeft = available.awaitNanos(delay);
                            nanos -= delay - timeLeft;
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }
    
最后需要提一下的是DelayQueue作为BlockingQueue，提供的drainTo方法用来把所有元素放入集合中，与toArray方法不同。toArray方法不保证元素按过期时间排序，而drainTo方法保证这一点，因为drainTo方法在方法内部循环调用了poll方法从PriorityQueue中取出了所有元素，并放入集合中。

完。