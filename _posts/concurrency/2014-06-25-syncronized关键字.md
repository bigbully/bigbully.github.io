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








