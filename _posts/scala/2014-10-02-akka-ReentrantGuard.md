---
layout: post
title: akka-ReentrantGuard笔记
description: 
category: scala
---

akka2.3.6

在读akka源码时看到一种使用可重入锁的方式：ReentrantGuard，忍不住记在这里。因为传统try-finally的使用方式太反人类了！

	final class ReentrantGuard extends ReentrantLock {

      @inline
      final def withGuard[T](body: ⇒ T): T = {
        lock()
        try body finally unlock()
      }
    }
    
ReentrantGuard实现代码就不用解释了。

如何使用看到这里也应该很清楚了：

	private val lock = new ReentrantGuard
	
	lock withGuard {
	  doSometing	
	}	
	
这多自然！	