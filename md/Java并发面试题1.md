---
title: Java并发面试题1
date: 2021-07-12
comments: true #是否可评论
toc: true #是否显示文章目录
categories: "面试题" #分类
tags:   #标签
    - JAVA
    - 面试题
---
### Java内存模型 (JMM)
    JMM用来屏蔽各种硬件和操作系统的内存访问差异，以实现让java程序在各种平台下都能达到一致的内存访问效果。
    JMM的主要目标是定义程序中各个变量的访问规则，即在JVM中将变量存储到内存和从内存中取出变量这样的底层细节。
    JMM规定了所有的变量都存储在主内存，每一条线程都还有自己的工作内存，线程的工作内存保存了被该线程使用到的
    变量的主内存副本拷贝，线程对变量的所有操作都必须在工作内存中进行，而不能直接读写主内存中的变量。

> 1. 请一下你对volatile关键字的理解？
    当一个变量被volatile修饰后，第一保证此变量对所有线程的可见性，当一个线程修改了这个变量的值，新值对于
    其他线程就是立即可见的。
    实现的原理是缓存一致性协议，修改完的数据需要同步到主内存去，这个数据要经过消息总线，其他cpu监控总线上的消息，
    如果这个volatile的值被更新，则自己的缓存中的相对应的值就变成无效，后再用的时候就需要从主内存中拉取最新的值。
    第二个语义是禁止指令重排序的优化，实现的原理是他会在操作前加一个lock前缀，这相当于一个内存屏障，指令
    重排序时不允许把后面的指令排序到内存屏障之前）。
    它不能保证原子性，原子性方面还是需要synchronized的配合使用。

> 2. 请讲一下你对synchronized关键字的理解？
    它可以实现可见性，原子性和有序性。
    它经过编译后，会在同步块的前后分别形成monitorenter 和 monitorexit这两个字节码指令，这两个字节码指令需要一个
    reference类型的参数来表明锁定和解锁的对象。如果没有指定，取决于它修饰的对象，取类还是类的实例作为锁对象。
    monitorenter指令获取对象的锁是可重入的，锁计数器可增减，减到0就是锁释放了。
    它和JUC中的可重入锁在新的1.6以后的版本中性能几乎无异，但是可重入锁有以下特点：
    1. 等待可中断 持有锁的线程长期不释放锁的时候，正在等待的线程可以选择放弃等待。
    2. 公平锁 多个线程在等待同一个锁时，必须按照申请锁的时间顺序来依次获取锁，synchronized和默认参数的可重入锁都是
    非公平锁，非公平锁的效率比较高。
    3. 锁可以绑定多个条件 使用newCondition()方法即可。
    为什么说Synchronized是重量级锁？
    因为它使用monitor对应的底层操作系统的mutex lock,操作系统实现线程切换需要从用户态转到内核态，这个成本非常高，状态
    之间的转换需要很长的时间。依赖操作系统的mutex lock实现的锁就是重量级锁。
    后面jdk对synchronized做了优化，无锁，偏向锁，轻量级锁在一定程度上优化了它的效率，但并发情况下还是难免会升级成重
    量级锁的。

> 3. 请讲一下你理解的ThreadLocal 变量 有什么用？
    我理解的场景，譬如有一个变量，你可能在多个类中都会访问，但是这些类都在一个线程中执行，那你就可以把这个变量放到ThreadLocal里面，
    这样非常方便访问。
    这个关键字的作用是什么?什么场景下会使用？它为每一个线程维护了变量副本，实现了线程安全。
    这种场景，譬如web服务中的controller等就可以使用这种，因为它们都在同一个线程中被访问。
    最常用的场景是session管理 ，数据库的连接对象。
    spark的TaskContext 在创建每一个task的时候都需要这个参数，在多线程并发的情况下，设置成ThreadLocal变量，可不互不干扰。
    val taskContext: ThreadLocal[TaskContext] = new ThreadLocal[TaskContext]
    它是怎么实现的呢？
    每一个线程都有一个类型为ThreadLocalMap的成员变量threadLocals，它是由ThreadLocal<>来维护的。
    它的Key是threadLocal对象，它的值就是set时传入的参数值。

> 4. 聊一下你对线程池的理解？你工作中怎么使用的？
     工作中是使用ExecutorService获取Executors返回的线程池，常用的是固定大小的线程池或含有调度功能的线性池。
     它又是调用的ThreadPoolExecutors。
     你提到的ThreadPoolExecutor它有哪些参数？
     （corePoolSize,MaxmiumPoolSize,KeepAlive, TimeUnit, workQueue)
      corepoolsize代表线程池中线程数量，MaximumPoolSize代表线程池中最大的线程数量。
      workQueue:表示任务队列，存储提交的未被执行的任务。
      它的工作流程如下：
      1. 新任务来的时候，如果正在运行的线程数量小于corepoolsize,则立刻创建新线程运行该任务。
      2. 如果正在运行的线程数量大于等于corepoolsize,则任务被添加到workqueue.
      3. 如果workqueue满了，而且正在运行的线程数量小于maximumpoolsize的话，还是要创建非核心线程来运行此任务。
      4. 如果workqueue满了，而且正在运行的线程数量大于或等于maximumpoolsize的话，根据拒绝策略处理。
      5. 如果一个线程无事可做，超过一定的时间，线程池会判断，如果运行的线程数量大于corepoolsize,则它会被停掉，
         即会慢慢地回到corepoolsize的大小。
      请讲一下它有哪些拒绝策略呢？
      1. AbortPolicy: 直接抛出异常，阻止系统正常运行。
      2. CallerRunPolicy: 在调用者线程中运行，这样会影响任务提交线程的运行情况。
      3. DiscardOldestPolicy: 抛弃工作队列中最前面的作业。
      4. DiscardPolicy: 抛弃当前新加来的作业。

>  5. 谈谈你对JUC中线程同步的几个数据结构？
      **CountDownLatch** 它可以实现类似计数器的功能，如果有一个作业需要四个线程完成后才可以继续，则可以使用这个
      CountDownLatch来完成。调用countDown方法来挂起当前线程。
      **CyclicBarrier** 它可以让一组线程等待到同一状态后再同时执行，当所有等待线程释放锁之后，它是可以被重用的。
      通过await()方法来挂起当前线程，当所有的线程都执行完此方法后可同时执行后面的代码。
      **Semaphore** 信号量，可以控制同时访问共享资源的线程数量，常用于线程池，连接池等实现。使用accquire()方法
      来获取一个资源，没有就等待，使用release()方法来释放一个资源。

>  6. 谈谈你对CAS的理解？
      CAS(COMPARE AND SWAP) 硬件层面支持这个指令后，JVM使用CAS来实现乐观锁。
      它接三个参数，要更新的变量，旧的值，新值。当多个线程都用CAS来更新一个变量值的时候，只有一个会胜出，失败的仅
      告知失败，但会继续尝试，不会挂起。
      CAS遭遇到ABA问题，可以通过版本号或时间来标记来解决这个问题，stampedReference。
      它不可以保证代码块的原子性，只能保证一个操作的原子性。
      它循环重试的过程中一直使用CPU资源。

>  7. 谈谈你对AQS的理解？
      它定义了一套多线程访问共享资源的同步器框架，许多同步类的实现都依赖于它，如semaphore,countdownlatch等。
      它维护了一个volatile int state（代表共享资源）和一个FIFO的线程队列（争用资源被阻塞时进入此队列），基于
      它可以设计独占锁或者是共享锁。
      它只是一个框架，具体资源的获取和释放，资源的数量都由具体类去实现。
      它有tryAccuire,tryRelease和tryAccureshare,Tryreleaseshare方法来获取和释放资源 。



