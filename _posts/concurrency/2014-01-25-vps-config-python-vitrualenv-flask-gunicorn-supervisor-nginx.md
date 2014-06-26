---
layout: post
title: synchronized关键字与我
description: 总结一下synchronized使用场景
category: concurrency
---
之前去小微金服面试，被问到是否熟悉并发问题，我非常谦虚的回答：稍微了解一点。但心想并发编程实战都翻了好几遍了，消息中间件也还做着，并发还能不熟悉？结果人家面试官就问了这么个问题：

	class Hesiod {
    	synchronized public void methodA(){
     	   System.out.println("methodA");
	    }

    	synchronized public void methodB(){
        	System.out.println("methodB");
	    }	
	}
	
	Hesiod h = new Hesiod();

当线程a调用对象h的methodA方法时，线程b调用对象h的b方法，会不会阻塞。








