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

在构造函数中创建了一个同步器Sync.这个sync继承了AbstractQueuedSynchronizer, 并完成了CountDownLatch的所有工作.

	private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        public int tryAcquireShared(int acquires) {
            return getState() == 0? 1 : -1;
        }

        public boolean tryReleaseShared(int releases) {
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
	
从源码中可以看到由CountDownLatch的构造函数传入的count被设置为sync的state,state在AbstractQueuedSynchronizer内部是volatile修饰的,它的状态改变对所有线程可见.

AbstractQueuedSynchronizer会强制他的子类实现tryAcquire(或tryAcquireShared),tryRelease(或tryReleaseShared)方法,从名称上可以看出,分别代表独占式和共享式.在这里CountDownLatch的Sync是共享式的同步器.

先看一下CountDownLatch的await方法是如何实现的.当某个线程CountDownLatch调用await方法时,会在这里阻塞,等待CountDownLatch调用countDown方法的次数超过count值,才被唤醒.
	
	public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

		public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }

acquireSharedInterruptibly方法是在AbstractQueuedSynchronizer已经实现好的.首先他会检查当前线程是否已经被中断, 然后调用用户自行实现的tryAcquireShared方法检查是否满足状态,tryAcquireShared方法的返回值为int类型.方法规定:

>**返回值**

> - 大于0.表示本次尝试获取锁成功,并且后续的其他线程再次尝试获取锁仍然有可能成功(后续的其他线程需要去检查是否能获取锁)
> - 等于0.表示本次尝试成功,但后续的其他线程不会成功获取锁.
> - 小于0.表示本次尝试失败.

再来看一下CountDownLatch实现的tryAcquireShared方法体:

	public int tryAcquireShared(int acquires) {
            return getState() == 0? 1 : -1;
    }

可以看到只有getState(),也就是构造函数中的count值为0时,才会返回1,即能够获取锁,并且后续的其他线程再次尝试获取锁仍然有可能成功.反之,则失败.

当CountDownLatch从未countDown过,自然在await时会失败,继而调用doAcquireSharedInterruptibly方法.

	private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED);
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    break;
            }
        } catch (RuntimeException ex) {
            cancelAcquire(node);
            throw ex;
        }
        // Arrive here only if interrupted
        cancelAcquire(node);
        throw new InterruptedException();
    }

认真分析一下这个方法.

 1. 首先创建共享类型的node并加入到FIFO队列中
 2. 获取node的前驱节点p
 3. 如果前驱节点p为队列头,则再次尝试获取锁tryAcquireShared
 4. 如果获取锁成功,则执行setHeadAndPropagate,用来把当前node设置为头结点,并向后传播自己获取锁成功的信息.这里需要详细解释一下为什么要向后传播.还记得之前提到过的共享模式定义吗?

Welcome to StackEdit!
===================


Hey! I'm your first Markdown document in **StackEdit**[^stackedit]. Don't delete me, I'm very helpful! I can be recovered anyway in the **Utils** tab of the <i class="icon-cog"></i> **Settings** dialog.

----------


Documents
-------------

StackEdit stores your documents in your browser, which means all your documents are automatically saved locally and are accessible **offline!**

> **Note:**

> - StackEdit is accessible offline after the application has been loaded for the first time.
> - Your local documents are not shared between different browsers or computers.
> - Clearing your browser's data may **delete all your local documents!** Make sure your documents are synchronized with **Google Drive** or **Dropbox** (check out the [<i class="icon-refresh"></i> Synchronization](#synchronization) section).

#### <i class="icon-file"></i> Create a document

The document panel is accessible using the <i class="icon-folder-open"></i> button in the navigation bar. You can create a new document by clicking <i class="icon-file"></i> **New document** in the document panel.

#### <i class="icon-folder-open"></i> Switch to another document

All your local documents are listed in the document panel. You can switch from one to another by clicking a document in the list or you can toggle documents using <kbd>Ctrl+[</kbd> and <kbd>Ctrl+]</kbd>.

#### <i class="icon-pencil"></i> Rename a document

You can rename the current document by clicking the document title in the navigation bar.

#### <i class="icon-trash"></i> Delete a document

You can delete the current document by clicking <i class="icon-trash"></i> **Delete document** in the document panel.

#### <i class="icon-hdd"></i> Export a document

You can save the current document to a file by clicking <i class="icon-hdd"></i> **Export to disk** from the <i class="icon-provider-stackedit"></i> menu panel.

> **Tip:** Check out the [<i class="icon-upload"></i> Publish a document](#publish-a-document) section for a description of the different output formats.


----------


Synchronization
-------------------

StackEdit can be combined with <i class="icon-provider-gdrive"></i> **Google Drive** and <i class="icon-provider-dropbox"></i> **Dropbox** to have your documents saved in the *Cloud*. The synchronization mechanism takes care of uploading your modifications or downloading the latest version of your documents.

> **Note:**

> - Full access to **Google Drive** or **Dropbox** is required to be able to import any document in StackEdit. Permission restrictions can be configured in the settings.
> - Imported documents are downloaded in your browser and are not transmitted to a server.
> - If you experience problems saving your documents on Google Drive, check and optionally disable browser extensions, such as Disconnect.

#### <i class="icon-refresh"></i> Open a document

You can open a document from <i class="icon-provider-gdrive"></i> **Google Drive** or the <i class="icon-provider-dropbox"></i> **Dropbox** by opening the <i class="icon-refresh"></i> **Synchronize** sub-menu and by clicking **Open from...**. Once opened, any modification in your document will be automatically synchronized with the file in your **Google Drive** / **Dropbox** account.

#### <i class="icon-refresh"></i> Save a document

You can save any document by opening the <i class="icon-refresh"></i> **Synchronize** sub-menu and by clicking **Save on...**. Even if your document is already synchronized with **Google Drive** or **Dropbox**, you can export it to a another location. StackEdit can synchronize one document with multiple locations and accounts.

#### <i class="icon-refresh"></i> Synchronize a document

Once your document is linked to a <i class="icon-provider-gdrive"></i> **Google Drive** or a <i class="icon-provider-dropbox"></i> **Dropbox** file, StackEdit will periodically (every 3 minutes) synchronize it by downloading/uploading any modification. A merge will be performed if necessary and conflicts will be detected.

If you just have modified your document and you want to force the synchronization, click the <i class="icon-refresh"></i> button in the navigation bar.

> **Note:** The <i class="icon-refresh"></i> button is disabled when you have no document to synchronize.

#### <i class="icon-refresh"></i> Manage document synchronization

Since one document can be synchronized with multiple locations, you can list and manage synchronized locations by clicking <i class="icon-refresh"></i> **Manage synchronization** in the <i class="icon-refresh"></i> **Synchronize** sub-menu. This will let you remove synchronization locations that are associated to your document.

> **Note:** If you delete the file from **Google Drive** or from **Dropbox**, the document will no longer be synchronized with that location.

----------


Publication
-------------

Once you are happy with your document, you can publish it on different websites directly from StackEdit. As for now, StackEdit can publish on **Blogger**, **Dropbox**, **Gist**, **GitHub**, **Google Drive**, **Tumblr**, **WordPress** and on any SSH server.

#### <i class="icon-upload"></i> Publish a document

You can publish your document by opening the <i class="icon-upload"></i> **Publish** sub-menu and by choosing a website. In the dialog box, you can choose the publication format:

- Markdown, to publish the Markdown text on a website that can interpret it (**GitHub** for instance),
- HTML, to publish the document converted into HTML (on a blog for example),
- Template, to have a full control of the output.

> **Note:** The default template is a simple webpage wrapping your document in HTML format. You can customize it in the **Advanced** tab of the <i class="icon-cog"></i> **Settings** dialog.

#### <i class="icon-upload"></i> Update a publication

After publishing, StackEdit will keep your document linked to that publication which makes it easy for you to update it. Once you have modified your document and you want to update your publication, click on the <i class="icon-upload"></i> button in the navigation bar.

> **Note:** The <i class="icon-upload"></i> button is disabled when your document has not been published yet.

#### <i class="icon-upload"></i> Manage document publication

Since one document can be published on multiple locations, you can list and manage publish locations by clicking <i class="icon-upload"></i> **Manage publication** in the <i class="icon-provider-stackedit"></i> menu panel. This will let you remove publication locations that are associated to your document.

> **Note:** If the file has been removed from the website or the blog, the document will no longer be published on that location.

----------


Markdown Extra
--------------------

StackEdit supports **Markdown Extra**, which extends **Markdown** syntax with some nice features.

> **Tip:** You can disable any **Markdown Extra** feature in the **Extensions** tab of the <i class="icon-cog"></i> **Settings** dialog.

> **Note:** You can find more information about **Markdown** syntax [here][2] and **Markdown Extra** extension [here][3].


### Tables

**Markdown Extra** has a special syntax for tables:

Item     | Value
-------- | ---
Computer | $1600
Phone    | $12
Pipe     | $1

You can specify column alignment with one or two colons:

| Item     | Value | Qty   |
| :------- | ----: | :---: |
| Computer | $1600 |  5    |
| Phone    | $12   |  12   |
| Pipe     | $1    |  234  |


### Definition Lists

**Markdown Extra** has a special syntax for definition lists too:

Term 1
Term 2
:   Definition A
:   Definition B

Term 3

:   Definition C

:   Definition D

	> part of definition D


### Fenced code blocks

GitHub's fenced code blocks are also supported with **Highlight.js** syntax highlighting:

```
// Foo
var bar = 0;
```

> **Tip:** To use **Prettify** instead of **Highlight.js**, just configure the **Markdown Extra** extension in the <i class="icon-cog"></i> **Settings** dialog.

> **Note:** You can find more information:

> - about **Prettify** syntax highlighting [here][5],
> - about **Highlight.js** syntax highlighting [here][6].


### Footnotes

You can create footnotes like this[^footnote].

  [^footnote]: Here is the *text* of the **footnote**.


### SmartyPants

SmartyPants converts ASCII punctuation characters into "smart" typographic punctuation HTML entities. For example:

|                  | ASCII                        | HTML              |
 ----------------- | ---------------------------- | ------------------
| Single backticks | `'Isn't this fun?'`            | 'Isn't this fun?' |
| Quotes           | `"Isn't this fun?"`            | "Isn't this fun?" |
| Dashes           | `-- is en-dash, --- is em-dash` | -- is en-dash, --- is em-dash |


### Table of contents

You can insert a table of contents using the marker `[TOC]`:

[TOC]


### MathJax

You can render *LaTeX* mathematical expressions using **MathJax**, as on [math.stackexchange.com][1]:

The *Gamma function* satisfying $\Gamma(n) = (n-1)!\quad\forall n\in\mathbb N$ is via the Euler integral

$$
\Gamma(z) = \int_0^\infty t^{z-1}e^{-t}dt\,.
$$

> **Tip:** To make sure mathematical expressions are rendered properly on your website, include **MathJax** into your template:

```
<script type="text/javascript" src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS_HTML"></script>
```

> **Note:** You can find more information about **LaTeX** mathematical expressions [here][4].


### UML diagrams

You can also render sequence diagrams like this:

```sequence
Alice->Bob: Hello Bob, how are you?
Note right of Bob: Bob thinks
Bob-->Alice: I am good thanks!
```

And flow charts like this:

```flow
st=>start: Start
e=>end
op=>operation: My Operation
cond=>condition: Yes or No?

st->op->cond
cond(yes)->e
cond(no)->op
```

> **Note:** You can find more information:

> - about **Sequence diagrams** syntax [here][7],
> - about **Flow charts** syntax [here][8].

### Support StackEdit

[![](https://cdn.monetizejs.com/resources/button-32.png)](https://monetizejs.com/authorize?client_id=ESTHdCYOi18iLhhO&summary=true)

  [^stackedit]: [StackEdit](https://stackedit.io/) is a full-featured, open-source Markdown editor based on PageDown, the Markdown library used by Stack Overflow and the other Stack Exchange sites.


  [1]: http://math.stackexchange.com/
  [2]: http://daringfireball.net/projects/markdown/syntax "Markdown"
  [3]: https://github.com/jmcmanus/pagedown-extra "Pagedown Extra"
  [4]: http://meta.math.stackexchange.com/questions/5020/mathjax-basic-tutorial-and-quick-reference
  [5]: https://code.google.com/p/google-code-prettify/
  [6]: http://highlightjs.org/
  [7]: http://bramp.github.io/js-sequence-diagrams/
  [8]: http://adrai.github.io/flowchart.js/
