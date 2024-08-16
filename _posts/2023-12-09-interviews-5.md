---
title: '面试相关（4）'
author: litao
date: 2023-12-09 21:32:00 +0800
categories: [interviews]
tags: [interviews]






---

### 🏷️ Android Handler相关

1. 为什么Loop是一个死循环不会阻塞主线程

   主线程本来就是处理我们的各种交互操作，当**MessageQueue**中没有消息时，也代表我们没有任何操作，也谈不上阻塞了。阻塞（anr）的前提条件是MessageQueue中有消息，而且改消息的执行时间太长，影响到了后续消息的执行才会阻塞。

2. 主线程为什么不用对Looper进行初始化

   ActivityThread初始化时以及进行了Looper初始化。

3. Message如何关联自己的handler

   msg有一个target参数，target就是所属的handler，分发消息就是直接调用的msg.target.dispatchMessage

4. Handler是怎么切换线程的呢

   发送消息最终是在Looper中调用msg.target.dispatchMessage来进行发送的，所以并没有发生线程的切换，而且取决于Looper的创建在那个线程，在主线程创建就会发送会主线程，在自线程创建就会发送回子线程。

5. Handler中post和sendMessage有什么区别

   post时是创建一个message同时将post的runnable作为回调添加进去，handler在收到消息时如果回调不为空就会直接执行这个回调。

   sendMessage是由我们自己创建一条消息直接发送，handler在收到消息时回调为空，会校验是否有设置callback有的话直接发送当前消息。

6. 获取Message实例的方式？为什么要用obtain的方式获取？

   可以同过 Message.obtain()，Handler.obtatinMessage()，new Message来来创建消息

   通常应该避免直接new Message，因为Message是有一个服用机制。

7. 如何保证线程和Looper的对应关系的？

   Looper在prepare的时候会将自身放到ThreadLocal中，通过Looper.myLooper()取值也是通过内部静态的sThreadLocal来获取，key是当前线程，从何保证了对应关系，每个线程的唯一性。

8. 同步屏障

   MessageQueue中取消息只能从头部取，添加消息是按先后顺序进行排序的，如果有一条消息需要立即执行，就需要开启同步屏障，内部开启的方式是使用MessageQueue.postSyncBarrier,会插入一条target为空的消息（使用者不可以直接创建target为null的消息，内部做了判断，会抛出异常），这个消息之后的普通消息会被屏蔽，优先执行异步消息，postSyncBarrier会返回一个int值的token，消息执行完后需要根据这个token来撤销屏障。

   如果想发送异步消息可以在创建**Handler**时使用携带async的构造，也可以在消息上直接设置为异步**setAsynchronous**

9. Handler消息延迟如何发送延迟消息

   通过sendMessageDelayed发送，会携带一个延迟时间参数，这个参数最终会结合当前时间赋值给message的when参数，MessageQueue获取next的时候会校验when参数，如果小于当前时间就会阻塞当前线程，每当有新消息插入又会立即唤醒。

10. MessageQueue 中的IdleHandler

    MessageQueue中的一个接口，当消息队列中没有任何消息，进入阻塞状态时，会回调此接口，我们可以通过Looper的getQueue方法来获取MessageQueue进而通过addIdleHandler添加监听回调。使用结束后需要移除。

11. Handler内存泄漏

    主要原因是非静态内部类，或者匿名内部类持有外部类引用照成

    引用链：activity->handler->Message->MessageQueue->Looper->Threadlocal->主线程->gc root

12. 阻塞唤醒机制

    MessageQueue中的数量为0  或者有消息但还没有达到执行时间，Looper会进入阻塞状态，等到下一个消息进入后 或 移除同步屏障后会再此唤醒开始处理新消息。

    

### 🏷️ Android WorkManager
主要为了处理一下几种持久性任务
- 立即执行的耗时较短的任务
- 长时间允许的任务
- 延时执行或周期性执行的任务

#### 工作原理
首先任务提交后WorkRequest详细后保存到数据库，根据约束条件从数据库拿出信息，构造好Worker,然后在Executor(默认单线程)中执行doWork，内部会根据优先级，依赖关系，系统内存状态，电量等一系列条件来决定何时执行。

#### 使用方式

1. 首先我们继承Worker实现 doWork方法创建任务，编写具体的任务逻辑，返回任务的执行状态，成功、失败、重试等。
2. 利用WorkRequestBuilder构建worker请求。
3. 使用WorkManager单例将任务请求加入到任务队列中。

#### 操作方法

1. 在执行Worker都是可观察的，我们可以通过worker的 id 、tag 并可以结合一些复杂条件（如：运行状态，名称等多条件组合）来获取当前的worker来进一步操作
2. 可以构建worker的工作链，让worker按指定顺序有序的执行。
3. 可以在worker内部指定工作进度。

### 🏷️ Android Activity的启动模式

1. **standard**：这是默认的启动模式，每次启动会创建一个新实例，加入栈顶
2. **singleTop**：如果目标Activity已经位于任务栈的顶部，则不会创建新的实例，而是复用顶部的现有实例，并调用它的`onNewIntent()`方法。如果目标Activity不在栈顶，则会创建新的实例
3. **singleTask**：系统在启动该Activity时，会检查任务栈中是否已经存在该Activity的实例。如果存在，则系统会将该Activity置于栈顶，并销毁其上的所有Activity，`onNewIntent()`会被调用以处理新的Intent，如果任务栈中不存在该Activity，则创建新的实例
4. **singleInstance**：类似于singleTask，但更为特殊。系统为该Activity创建一个单独的任务栈，并且在整个系统中，这个Activity在该任务栈中只能存在一个实例，后续打开新的Intent都会通过onNewIntent()。

