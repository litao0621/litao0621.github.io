---
title: '面试相关（1）'
author: litao
date: 2023-12-02 15:34:00 +0800
categories: [interviews]
tags: [interviews]


---

### 🏷️  图片占据的真实内存

通常所占据的内存大小为 分辨率*像素大小，来确定所占据的内存大小，像素大小需要根据色彩模式来确定包括

- ALPHA_8 -- (1B)
- RGB_565 -- (2B)
- ARGB_4444 -- (2B)
- ARGB_8888 -- (4B)
- RGBA_F16 -- (8B)

但实际使用会发现，同一张图片在不同设备，加载来源不同的情况下，实际的内存大小不相同，大部分情况是由于我们使用了`Bitmap.decodeResource()`，因为内部在加载图片时会根据当前图像所在文件夹对应的dpi和设备dpi进行一次转化。转换后的宽高发生了变化

- 新高度 = 原高度 * (设备 dpi / res 目录 dpi )
- 新宽度 = 原宽度 * (设备 dpi / res 目录 dpi )

### 🏷️  Kotlin 扩展函数的实现原理
就是在当前类中以扩展的对象作为第一个参数生成的静态方法

### 🏷️ Kotlin 伴生对象的创建是懒汉还是饿汉模式

饿汉模式，反编译后可以看到Companion类在类初始化就被创建了，定义为了static final

### 🏷️ Android常用存储方式

1. SharedPreferences

   最传统的方式，适合存储少量数据，因为SP会将xml文件读到内存中，所有尽量不要去定义特别大的key  or  value

   - 优点：会自动备份，不丢数据
   - 缺点：不支持多进程；会引发ANR(commit时会阻塞调用线程，apply时再页面暂停销毁时也会阻塞主线程进行存储，从而导致ANR); 还有就是apply的成功与否无法回调。

2. DataStore

   DataStore是官方为了处理SharedPreferences的一系列缺点产生的

   - 优点：综合角度来看性能高于SharedPreferences（但会根据数据量的大小出现一些差异）；支持多进程
   - 缺点：由于DataStore是基于协程+Flow实现的，对纯java开发者并不友好

3. MMKV

   这是非官方大家使用较多的方案，号称性能极高

   - 优点：官方给出的测试结果是写入极快，性能高（但这个结果是在一定条件下，同步情况下与上面的方式对比差异确实很多，但异步情况下差异就会很小了，而且如果数据量较大时可能还会出现比上面方式更差的情况。）；支持多进程。
   - 缺点：由于存储的底层机制，在系统关机操作系统的崩溃情况下可能会导致数据文件损坏，由于没有备份机制无法恢复。

4. 

### 🏷️ IntentService相关

`IntentService` 是一个在后台线程中处理异步任务的服务。与常规的 `Service` 不同，内部使用HandlerThread+Hanler，它自带了一个工作线程来处理所有传递给它的 `Intent`，并且在任务完成后会自动停止服务。

### 🏷️ Service的启动方式

- startService：会一直运行下去，需要外部调用了stopService()或stopSelf()方法时或系统回收，该Service才会停止运行并销毁。 生命周期会经历 onCreate()->onStartCommand()->onBind()->onDestory()
- bindService：这种方式启动的服务是与客户端（如Activity）绑定在一起的，一旦所有客户端都解绑，服务就会停止运行。 生命周期会经历 onCreate()->onBind()->onUnbind()->onRebind->onDestory()

### 🏷️  LayoutInflater.inflate参数说明

**`resource`**：表示要加载的 XML 布局资源的 ID。

**`root`**：表示生成的视图层次结构的根视图。

**`attachToRoot`**：`boolean` 类型，决定了是否将加载的布局文件附加到 `root` 中。

- `true`：将加载的视图添加到 `root` 中，并返回 `root` 作为结果视图。此时，加载的布局文件的根视图被附加到 `root`，并且会考虑 `root` 的布局参数（如 `LayoutParams`）。
- `false`：不将加载的视图附加到 `root` 中，直接返回加载的视图。虽然不会附加到 `root`，但加载的视图仍会使用 `root` 的布局参数。

需要注意 root的值会影响resource 的LayoutParams

### 🏷️  打开透明activity 下层页面的生命周期。

A to B : A :onPause()，B:onCreate()->onStart()->onResume()

B back A : B onPause() , A Resume() , B onStop() , B onDestroy()

### 🏷️  下拉状态栏是否影响下层activity生命周期

不影响，activity的生命周期改变需要有本身或其它activity的变化引起。











