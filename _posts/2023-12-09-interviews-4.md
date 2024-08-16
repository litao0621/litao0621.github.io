---
title: '面试相关（4）'
author: litao
date: 2023-12-09 19:13:00 +0800
categories: [interviews]
tags: [interviews]





---

### 🏷️ LiveData 实现原理及涉及的相关问题

Livedata 具有生命周期的感知能力，能够根据组件（Activity、Fragment等）的生命周期来更新数据，livedata是线程安全的。内部通过版本控制来准确的将数据分发给订阅者。livedata中存在一个版本，在set or post data时会version++，observer也存在一个版本,在接受到数据时也会version++，在每次进行数据通知时都会对比这两个版本，如果相同则不进行分发。

1. 如何感知到生命周期

在我们调用observer时需要传递LifecycleOwner，内部使用LifecycleOwner addObserver进行了生命周期的绑定，内部

2. setValue 和 postValue有什么区别

- setValue: 必须在主线程调用，内部对线程做了校验
- postValue: 可以在任意线程调用，synchronized mDataLock锁，保证了线程安全，最终通过handler讲数据发送会了主线程

3. set重复的值后订阅者会收到相同的值吗

会收到相同值，因为内部没有做相同数据的校验

4. Livedata的数据倒灌

因为内部是根据数据的版本和订阅者的版本做对比进行数据分发的，所以内部会存在一个类似粘性事件的原理，就是只要livedata内部有值，你无论何时订阅都会接收到最新的值。

如果需要防止倒灌可以使用官方提供的**SingleLiveEvent**方式，我们继承Livedata后添加一个状态来表示数据是否被消费过，只有未消费的才去通知。

5. Livedata会丢数据吗

在高频postValue时可能会丢，因为postValue时先将数据暂存，然后使用handler进行post ，之后才会调用setValue,这中间存在一定间隔，可能会导致未通知订阅者时值已经被覆盖的情况



### 🏷️ Java中的volatile关键字

`volatile`关键字是一个用于标记变量的类型修饰符，主要用于在多线程环境中确保变量的可见性和防止指令重排序

1. 可见性：java内存模型定义了所有变量都存储在主内存中，每个线程还有自己的工作内存，工作内存中使用到的变量是主内存中的拷贝，所有操作都是使用的工作内存中的数据，不同线程也无法直接相互访问，正常情况多个线程同时操作变量是，并不能保证每个工作线程中的缓存都会被立即更新，volatile就是为里解决这个问题，volatile修饰后，每个工作线程在使用volatile修饰后的变量时，都会强制从主存中进行更新，从而保证了可见性。

2. 防止指令重排：JVM和CPU是允许在不改变语意的情况下对程序中的指令进行重排，volatile修饰后则可以防止重排，让它按原有的顺序执行。

   例如：volatile其实可以理解为一个触发数据刷新的指令，当读取/写入volatile变量时同时也会出发其他变量的更新，我们有事会把volatile变量的读取放在最前，写入放在最后，用来同时更新其他变量，但如果这种情况下指令进行了重排后，可能就无法达到我们原有的效果。导致普通变量的读取被重排到了volatile变量的读取之前，或普通变量写入重排到了volatile变量写入之后。


### 🏷️ Java中的线程池
线程池的基本思想是预先创建若干线程，当有任务需要执行时，线程池会分配一个空闲线程来执行任务。如果所有线程都在忙碌，新的任务会进入等待队列，直到有空闲线程可用。
利用线程池的主要目的是为了提高性能，更好的管理资源，提高响应速度，简化编程模型。系统为我们提供了一下几种类型的线程池。

1. 固定线程池（FixedThreadPool）
     线程池的大小是固定的，当线程都在忙碌时，新线程会进入等待

2. 单线程池（SingleThreadExecutor）
     每次只会有一个线程执行任务，其他任务会在队列中等待

3. 缓存线程池（CachedThreadPool）

     线程池会根据需要创建新线程，并在空闲时回收线程。当任务较多时，缓存线程池可以动态调整线程数量

4. 调度线程池（ScheduledThreadPool）

     支持定时任务和周期性任务执行

5. 自定义线程池

     ``` java
     ThreadPoolExecutor customThreadPool = new ThreadPoolExecutor(
         5, // corePoolSize
         10, // maximumPoolSize
         60, // keepAliveTime
         TimeUnit.SECONDS, // unit
         new LinkedBlockingQueue<>(100) // workQueue
     );
     ```

     - corePoolSize: 核心线程数，线程池中始终存活的线程数
     - maximumPoolSize: 最大线程数，线程池中允许的最大线程数
     - keepAliveTime: 存活时间，线程没有任务执行时最多保持多久时间会终止
     - unit: keepAliveTime的时间单位
     - workQueue：一个阻塞队列，用来存储等待执行的任务，均为线程安全
     - threadFactory: 线程工厂，主要用来创建线程
     - handler：拒绝策略。活动线程数最大等于线程数，并且任务队列已满执行此策略

  

