---
layout: post
title: java内置锁
description: 总结一下synchronized使用场景
category: concurrency
---
之前去面试，面试官就问了这么个问题， 一个类的两个方法都用synchronized关键字来修饰，多个线程同时调这两个方法，会不会彼此阻塞？

这个问题可以用以下代码来表达：

	public class Hesiod {

	    public int i = 0;

    	synchronized public void increment(){
        	i++;
   		}

	    synchronized public void decrement(){
    	    i--;
	    }

	    public static void main(String[] args){
    	    final Hesiod h = new Hesiod();
        	final CountDownLatch latch = new CountDownLatch(2);
	        new Thread(){

    	        @Override
	            public void run() {
	                for (int i = 0; i < 10000; i++) {
	       	            h.increment();
	                }
	                latch.countDown();
	            }
	        }.start();

	        new Thread(){

	            @Override
	            public void run() {
	                for (int i = 0; i < 10000; i++) {
	                    h.decrement();
	                }
	                latch.countDown();
	            }
	        }.start();

	        try {
	            latch.await();
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        }
	        System.out.println(h.i);
	    }
	}

当两个线程同时开启，一个调用synchronized的increment方法，一个调用synchronized的decrement方法，最后结果仍然是：0.

这也就引出了synchronized的几种加锁方式:

	public class Sophocles {
    
    	//静态方法加锁实际上是加在Sophocles.class上
	    synchronized public static void methodA1(){
    	    
	    }	

		//加锁的范围与methodA1一致
    	public static void methodA2(){
	        synchronized (Sophocles.class){
            
    	    }
	    }
    
    	//修饰在类方法上得锁，实际上是加在当前对象上也就是this
	    synchronized public void methodB1(){
	        
	    }
    
    	//加锁的范围与methodB1一致
	    public void methodB2(){
	        synchronized (this){
            
	        }
	    }
    
	}
	
显然当时那个面试官问的问题指的两个方法都是在this上加锁，被多个线程同时调用的时候必然会阻塞。

不过内置锁还有一个“重入”的特性

	public class Heracles {
		synchronized public void methedSuper(){
		
		}
	}
	
	public class Theseus extends Heracles {
		synchronized public void methodChild(){
			super.methodSuper();
		}
	}
	
这段代码看上去是先调用了子类的methodChild，获得了this上得锁，然后又尝试去调用this上加锁的父类方法methodSuper。因为内置锁有“重入”的机制，如果某个线程师徒获得一个已经由它持有的锁，那么请求会成功。所以上面的场景并不会发生死锁。

由于内置锁直接加在方法上会严重影响性能，所以我认为本着降低锁的粒度初衷，显示的写明加锁的对象：

	synchronized (obj){
	
	}
	
能时刻提醒自己，加锁的粒度是不是还能进一步缩小。

关于可见性

比较在使用内置锁的时候容易忽视的一个含义在于可见性。如果想通过内置锁实现共享变量的同步，也就是确保所有线程都能看到共享变量的最新值，就必须使执行读操作或写操作的线程都在同一个锁上同步。

使用互斥锁实现两个线程间的搬运操作

	
	public class Jason {

    	private Object lock = new Object();
   		private Item item = null;//需要搬运的货物

	    public void start(){
    	    new Put().start();
        	new Get().start();
	    }


	    public static void main(String[] args){
    	    Jason jason = new Jason();
	        jason.start();
	    }

        //Put线程创建货物，搬运到指定位置，并通知Get线程去取货
	    class Put extends Thread {

    	    @Override
	        public void run() {
    	        for (int i = 0; i < 10000; i++) {
        	        synchronized (lock){
            	        item = new Item(i);
                	    System.out.println("put:" + item.i);
	                    lock.notifyAll();
    	                try {
        	                lock.wait();
            	        } catch (InterruptedException e) {
                	        e.printStackTrace();
	                 	}
	                }
    	        }
	        }
	    }
	    
	    //Get线程收到通知后，取货，再通知Put线程搬运新的货物
	    class Get extends Thread {

    	    @Override
	        public void run() {
	            while (true){
    	            synchronized (lock){
	                    if (item == null){
	                        try {
	                            lock.wait();
	                        } catch (InterruptedException e) {
	                            e.printStackTrace();
	                        }
	                    }
    	                Item myItem = item;
	                    item = null;
	                    System.out.println("get:" + myItem.i);
	                    lock.notifyAll();
	                }
	            }
	        }
	    }

    	class Item {
	        private int i;
	
    	    public Item(int i){
	            this.i = i;
	        }
	    }
	}

两个线程使用同一个synchronized内置锁，当一个线程等待时会释放持有的锁，从而实现两个线程协作搬运的功能。

顺便补充一下jdk1.6之后对于虚拟机对于锁的优化。

常规来说当多个线程想要获得同一个锁时，只有一个线程会获得，而其余的线程则会挂起，当轮到这个挂起的线程工作时，在恢复回工作状态。但这种操作会带来性能上的开销。

自旋锁

jdk1.6之后HotSpot虚拟机引入了锁的适应性自旋。什么意思呢？就是当多个线程争抢同一个锁时，未争抢到锁的线程不会马上挂起，而是使用自旋（死循环）+ cas不停地查看锁是不是已经释放。这种优化无疑加重了cpu的负担，但当遇到被加锁的代码块执行速度很快时，自旋次数不会很多，线程避免了挂起和恢复操作。当然线程不会无休止的自旋，当达到一个阈值之后，线程会正常挂起。自旋次数的默认值是10次，用户可以使用参数-XX:PreBlockSpin来更改。 

锁消除

锁消除值得便是HotSpot虚拟机的JIT编译器在发现一段代码虽然加了锁，但是实际上不可能被多个线程同时执行或者没有任何变量会被多个线程公用，则虚拟机会把锁消除。最简单的例子就是一个方法中的所有用到的变量都是方法的临时变量，这个方法或方法体内，是不用加锁的。

锁膨胀

事实上频繁的加锁解锁也是很消耗资源的，JIT当发现加锁的位置不合理的时候，可以扩大加锁范围，减少不必要的加锁解锁操作。最典型的例子是循环体内加锁，JIT会把锁膨胀到循环体外。

偏向锁

偏向锁，顾名思义就是加锁的过程偏袒第一次获得锁的线程。使用MarkWord中的固定位在第一次进入锁时使用cas记录线程id，如果这之后只有一个线程使用锁，那么不会再进行cas操作，也不会加锁解锁。直接进入同步代码块。

轻量级锁

轻量级锁能提升程序同步性能的依据是“对于绝大部分的锁，在整个同步周期内都是不存在竞争的”，这是一个经验数据。如果没有竞争，轻量级锁使用CAS操作MarkWord中的状态位避免了使用互斥量的开销，但如果存在锁竞争，除了互斥量的开销外，还额外发生了CAS操作，因此在有竞争的情况下，轻量级锁会比传统的重量级锁更慢。










