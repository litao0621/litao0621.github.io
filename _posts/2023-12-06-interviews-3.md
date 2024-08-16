---
title: '面试相关（3）'
author: litao
date: 2023-12-06 19:13:00 +0800
categories: [interviews]
tags: [interviews]




---

### 🏷️`ArrayList` 和 `LinkedList`区别及优缺点

#### 1.ArrayList

- 实现方式：基于动态数组实现，底层为一个数组，当需要增加更多元素时会将旧数组元素复制到新数组

- 扩容逻辑：初始大小为10，超出容量后会扩容到之前的1.5倍

- 优点：由于底层是数组，可以通过索引快速访问任意元素（时间复杂度O(1)）。由于数组是连续的内存，空间利用率也高。

- 缺点：由于是数组，插入删除需要移动大量元素，效率较低（时间复杂度0(n)）。 扩容时会重新创建更大的数组，并且需要复制元素，会带来一些开销。

#### 2.LinkedList

- 实现方式：基于双向列表实现，每个元素节点包含数据及前后两个指针指向前后数据。
- 优点：插入删除时只需要调整元素指针（时间复制度O (1)），效率较高，并且无需额外的扩容。
- 缺点：查询效率较低因为要遍历整个节点（时间复杂度O(n)），每个节点都需要额外空间来保存指针，空间利用率要低于`ArrayList`


🏷️`Hashmap` 和`Hashtable`区别及优缺点

#### 1. Hashmap
- 非线程安全，需要去手动同步（如使用 `Collections.synchronizedMap()` 或 `ConcurrentHashMap`）
- 性能由于未使用同步机制，要优于Hashtable
- 允许一个 `null` 键和多个 `null` 值。`HashMap` 允许存储 `null` 作为键和值。
- 内部使用列表实现，当链表长度超过一定阈值（默认8）时，链表会转换成红黑树，以提高查询效率。
- 容量默认的初始容量为 16，负载因子为 0.75。它在需要扩容时将容量翻倍，同时重新计算所有元素的哈希值并重新分配它们的位置，这会导致短暂的性能开销

#### 2. Hashtable
- 线程安全，内部大量使用了synchronized
- 由于同步机制的存在，性能要差于Hashmap,高并发下由于每次都获取锁，很容易造成瓶颈
- 不允许 `null` 键和 `null` 值。如果尝试插入 `null` 键或值，会抛出 `NullPointerException`。
- Hashtable内部也使用列表，但没有使用红黑树对hash冲突做进一步优化
- 默认的初始容量为 11，负载因子为 0.75。当需要扩容时，`Hashtable` 的容量增长为 `2 * 当前容量 + 1`。


### 🏷️ 环形列表的判断

1.快慢指针法（时间O(n)，空间(1)）

定义两个指针 `slow`,`fast`初始都指向列表的同一个元素，`slow`每次移动一步，`fast`每次移动两步，每次移动后检测两个元素是否相同，如果相同则存在环，如果任意一个指针出现null则无环。

2.哈希表遍历时间法（O(n)，空间(n)））

使用哈希表存储访问过的节点，如果再次遇到相同元素则存在环

3.修改列表结构法（时间O(n)，空间(1)）

在便利的同时修改元素的next节点为一个特殊值，如果再次遇到此特殊值时说明有环，由于会改变列表结构，只适用于判断或无需保留原始数据的情况。

3.递归法

传递当前节点和一个哈希表，递归调用中添加已遍历的元素，同时判断哈希表中是否包含当前元素，如果不包含且不为空则继续携带当前元素的next和哈希表继续调用。由于递归的栈空间开销，可能导致栈溢出问题。

### 🏷️ 强引用、软引用、弱引用和虚引用

- 强引用：强引用是 Java 中最常见的引用类型。当一个对象通过强引用被引用时，只要不主动释放垃圾收集器永远不会回收它。这是默认的引用类型

- 软引用：软引用是一种比强引用弱，但比弱引用强的引用类型。被软引用关联的对象在 JVM 内存不足时会被回收。软引用通常用于实现内存敏感的缓存，即当内存不足时，可以让垃圾收集器回收这些缓存对象。例如常用语图片的回收。
- 弱引用：弱引用是一种比软引用更弱的引用类型。被弱引用关联的对象只要垃圾收集器运行，不管内存是否充足，都会被回收。例如常用于易发生内存泄露的地方。
- 虚引用：仅用于一些内存管理监控，需与`ReferenceQueue`结合使用。

### 🏷️ java 中equals 和 hashcode相关问题

1. 重重写equals 必须重写hashcode吗

   如果我们在基于哈希的集合中（如：HashMap，HashSet）使用对象，那么需要重写hashcode,因为这些数据结构需要通过哈希码来快速定位对象。通常按照规范来说应该同时去重写，无论何种情况。

2. **equals** 和 == 的区别

   Obeject中默认是使用==进行比较，如果子类未重新则没有区别，通常我们会去重写，目的是为了比较内容相等。同时重写equals需要满足，自反性，对称性，传递性，一致性，非空性。

3. equals()已经完成了对比为什么还要有hashCode()？

   因为hashCode()并不是完全可靠，有时候不同的对象他们生成的hashcode也会一样

   - equals()相等的两个对象他们的hashCode()肯定相等，也就是用equals()对比是绝对可靠的。
   - hashCode()相等的两个对象他们的equals()不一定相等，也就是hashCode()不是绝对可靠的。

### 🏷️ java 内存模型

 1. 程序计数器（Program Counter Register）

    可以看作字节码的**行号指示器**，每个线程都有一个独立的程序计数器，是线程私有的。告诉程序按什么顺序去执行分支、跳转、循环、异常处理等。

2. jvm虚拟机栈（JVM Stack）

   管理Java方法的调用和执行，每个方法在执行时会创建一个栈帧（Stack Frame）用于存储局部变量以及方法执行过程的相关信息。是线程私有的

3. 本地方法栈（Native Method Stack）

   类似于jvm虚拟机栈，只不过是为本地方法提供的，简单说就是非 Java 代码的接口。是线程私有的

4. 堆（Heap）

   存储所有的对象实例和数组，是垃圾收集器管理的主要区域，线程共享，所有线程都可以访问

5. 方法区（Method Area）

   存储类的元数据、常量、静态变量、即时编译器编译后的代码等

6. 直接内存（Direct Memory）

   直接内存并不是JVM运行时数据区的一部分，但也被频繁使用。主要用于NIO（New Input/Output）中的直接缓冲区

### 🏷️ java gc相关

垃圾回收通常分为以下几种

- Minor GC (小型GC)：新生代GC事件
- Major GC（大型GC）：老年代GC事件,使用标记清除法。
- Full GC (完全GC)：整个堆的GC事件，通常主动调用System.gc或老年代空间不足或方法区空间不足时触发。

堆（Heap）是jvm管理的最大的一块空间，jvm会利用分代法进行垃圾的回收，堆中有一下几个区域

1. 新生代

   主要存放新创建的对象以及年龄较小的对象新生代中又被分为一下三个区域，大部分数据创建后首先会进入Edem区（对象占用内存较大的会直接进入老年代），Edem区内存不足时，每次发生Minor GC后 如果对象还存活就会进入幸存者区，之后每次Minor GC都会使存活的对象年龄+1，

   1. Edem（舒适区）：Edem区内存不足时会触发Minor GC
   2. Survivor from（幸存者1区）：from 于 to 会每次发生Minor GC发生互换，清空另一方
   3. Survivor to（幸存者2区）：

2. 老年代

   Minor GC的持续进行会使老年代中的数据也不断增加，当老年代空间不足时就会触发Major GC,首先从GC root进行遍历，把可达对象打上标记，再进行二次遍历，将没有标记的对象进行清除。

涉及到的清除算法

1. 引用计数法：所有引用对象会添加一个计数器，来表示被引用的次数。引用次数为0的说明没有其他引用，但无法处理循环引用的对象。
2. 根可达法：通常使用GC Roots作为起始对象开始搜索叶子结点，所有遍历到的对象进行标记。

3. 标记清除法：第一遍利用GC root 根可达法进行标记，第二次遍历未标记的对象进行清除，此过程发生在老年代
4. 复制清除法：第一遍利用GC root 根可达法进行巡渣存活的对象复制到另一区域，第二部删除所有对象，发生在年轻代的幸存者区域。

### 🏷️ 面向对象概述

#### 1. 三大特性

- 封装：尽量隐藏细节，仅提供给给与外界交互的功能接口
- 继承：尽量抽象公共特性，合理划分，子类拥有父类所有特性并且可以扩展自己的特性，如 生物 -> 动物  -> 狗 -> 泰迪
- 多态：

#### 1. 五个原则

- 单一责任原则：每个对象应该都有自己的职责，进行合理划分，减少各自职责的混淆
- 开放封闭原则：对象应该对扩展是开放的，对修改是封闭的
- 里氏替换原则：子类可以替换父类，并可以出现在父类可出现的任何地方
- 依赖倒置原则：高纬度的模块不应该依赖低维度的模块（生物不应该依赖动物）
- 接口隔离原则：接口的设计不应该强迫使用者去实现他用不上的接口，可以进行合理分类，拆解。

### 🏷️ Java中的锁

1. synchronized 锁

   可锁定当前实例方法，静态方法，代码块

2. ReentrantLock 锁

   是 `java.util.concurrent.locks` 包中提供的一个可重入锁，支持多个线程多次获取同一把锁；支持等待过程的响应中断；支持限时获取锁，超时则放弃锁；可以选择公平或非公平锁，公平锁保证等待时间最长的线程优先获得锁

3. ReentrantReadWriteLock 锁

   是一个读写锁，用于区分读操作和写操作。它允许多个线程同时进行读操作，但写操作是互斥的

4. StampedLock 锁

   它提供了比 `ReentrantReadWriteLock` 更高效的锁策略，特别是在读操作多、写操作少的场景中，可进行乐观读

### 🏷️ 动态代理，静态代理

1. 静态代理

   静态代理是指由开发者在代码中明确创建代理类，代理类实现与目标对象相同的接口，并在代理类的方法中调用目标对象的方法。代理类在编译时就已经确定，因此称为静态代理。使用简单，但灵活性较低，复用性也较低。

   ```java
   // 定义一个接口
   interface Service {
       void performTask();
   }
   
   // 目标类，实现了接口
   class RealService implements Service {
       @Override
       public void performTask() {
           System.out.println("Performing task in RealService");
       }
   }
   
   // 静态代理类，也实现了接口
   class StaticProxy implements Service {
       private final RealService realService;
   
       public StaticProxy(RealService realService) {
           this.realService = realService;
       }
   
       @Override
       public void performTask() {
           System.out.println("Before performing task");
           realService.performTask(); // 调用目标对象的方法
           System.out.println("After performing task");
       }
   }
   
   // 使用静态代理
   public class StaticProxyExample {
       public static void main(String[] args) {
           RealService realService = new RealService();
           Service proxy = new StaticProxy(realService); // 使用代理对象
           proxy.performTask();
       }
   }
   
   ```

2. 动态代理

   动态代理是在运行时通过反射机制动态生成代理类，而不是在编译时手动创建代理类。Java 提供了 `java.lang.reflect.Proxy` 类和 `InvocationHandler` 接口用于实现动态代理。与静态代理不同，动态代理可以代理任意接口，而不需要提前编写代理类。灵活性、复用性较高，但使用了反射有一定性能开销。

   ```java
   import java.lang.reflect.InvocationHandler;
   import java.lang.reflect.Method;
   import java.lang.reflect.Proxy;
   
   // 定义一个接口
   interface Service {
       void performTask();
   }
   
   // 目标类，实现了接口
   class RealService implements Service {
       @Override
       public void performTask() {
           System.out.println("Performing task in RealService");
       }
   }
   
   // 动态代理类，必须实现 InvocationHandler 接口
   class DynamicProxy implements InvocationHandler {
       private final Object target;
   
       public DynamicProxy(Object target) {
           this.target = target;
       }
   
       @Override
       public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
           System.out.println("Before performing task");
           Object result = method.invoke(target, args); // 调用目标对象的方法
           System.out.println("After performing task");
           return result;
       }
   }
   
   // 使用动态代理
   public class DynamicProxyExample {
       public static void main(String[] args) {
           RealService realService = new RealService();
           
           // 创建动态代理对象
           Service proxyInstance = (Service) Proxy.newProxyInstance(
                   realService.getClass().getClassLoader(),
                   realService.getClass().getInterfaces(),
                   new DynamicProxy(realService));
   
           proxyInstance.performTask(); // 使用代理对象调用方法
       }
   }
   
   ```

### 🏷️ Hashmap 原理
