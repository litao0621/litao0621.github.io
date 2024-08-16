---
title: '面试相关（2）'
author: litao
date: 2023-12-05 18:13:00 +0800
categories: [interviews]
tags: [interviews]



---

### 🏷️ Android Framework AMS

AMS主要用于管理应用程序、Activity的生命周期，包括启动、切换、调度、暂停和终止应用程序及其组件,用于保证应用程序资源的合理分配，同时AMS实现了Ibinder接口，主要用于系统中各服务进程间通信，包括PackageManagerService (PMS)、WindowManagerService (WMS)、ContentProvider、BroadcastReceiver等服务等通信,ASM主要职责包括

- **管理应用程序的生命周期**：包括启动、暂停、恢复和终止应用程序。

- **管理 Activity 的栈**：确保前台 Activity 的正确显示和后台 Activity 的保存状态。

- **进程和线程管理**：管理应用程序的进程和线程，确保系统资源的有效利用。

- **内存管理**：在内存不足时，终止不必要的进程和活动以释放资源。

- **广播（Broadcast）的调度**：处理和调度系统广播和应用程序广播。

- **服务（Service）的管理**：启动、停止和调度服务。

- **内容提供者（Content Provider）的管理**：管理应用程序之间的数据共享。

###  工作流程

1. 应用启动：当应用启动时，系统会通过 `startActivity` 方法向 AMS 发送请求，AMS 会检查请求的合法性，并在需要时创建一个新的task来包含新的 Activity，如果应用程序进程没有启动，AMS 会请求 Zygote 进程来启动一个新的进程，如果已经启动，AMS 会通过 Binder 通信将启动 Activity 的请求发送到目标应用程序的 ActivityThread

2. Activity的调度：AMS 负责调度前台和后台的 Activity，当用户切换到另一个应用程序时，AMS 会将当前 Activity 移到后台，并启动新的前台 Activity，同时管理 Activity 的栈，确保 Activity 的正确状态（如启动、暂停、停止和销毁）
3. 内存管理：AMS 与 Low Memory Killer和 OOM 机制合作，监控系统内存使用情况。在内存不足时，AMS 会选择性地终止不必要的进程和活动以释放资源。

#### 与其他系统组件的交互

AMS 会与一下组件密切交互

- **PackageManagerService (PMS)**：用于管理应用程序的安装、卸载和查询操作。
- **WindowManagerService (WMS)**：用于管理窗口和显示内容，确保前台 Activity 的正确显示。
- **ContentProvider**：AMS 管理应用程序之间的数据共享，确保数据的一致性和安全性。
- **BroadcastReceiver**：AMS 负责调度和处理系统广播和应用程序广播，确保消息的及时传递。

#### 内部结构

ActivityStack ->TaskRecord -> ActivityRecord



### 🏷️ Android Framework WMS

WMS负责管理所有窗口（包括应用窗口和系统窗口）的显示、布局和交互。WMS 确保不同窗口之间的正确排列和绘制，并处理窗口的创建、删除和更新等操作。

- **窗口的创建和删除**：管理应用程序和系统窗口的创建和删除。

- **窗口的布局和层次**：管理窗口的层次结构和布局，包括前台窗口、后台窗口和系统窗口。

- **窗口的绘制**：协调窗口内容的绘制和刷新。

- **输入事件处理**：分发输入事件（如触摸和键盘事件）到对应的窗口。

- **窗口动画**：管理窗口的打开、关闭和切换动画。

- **屏幕旋转**：处理屏幕旋转和窗口的重新布局。

#### 工作流程

1. 窗口的创建： 应用程序通过 `WindowManager` 请求创建窗口，WMS 处理该请求并创建相应的 `WindowState` 对象。
2. 窗口的布局: WMS 负责计算窗口的位置和大小，并请求窗口的绘制。
3. 窗口的绘制: WMS 与 SurfaceFlinger 协作，将窗口内容绘制到屏幕上。
4. 输入事件处理: WMS 处理输入事件（如触摸和键盘事件），并将其分发到对应的窗口。

#### WMS 与其他系统组件的交互

- **ActivityManagerService (AMS)**：AMS 负责管理应用程序的生命周期，WMS 负责管理应用程序的窗口，两者协作确保窗口的正确显示。
- **InputManager**：WMS 负责管理窗口的输入事件，InputManager 负责具体的事件处理和分发。
- **SurfaceFlinger**：WMS 与 SurfaceFlinger 协作进行窗口的绘制，SurfaceFlinger 负责实际的绘制操作。
- **DisplayManagerService (DMS)**：WMS 通过 DMS 管理多显示屏上的窗口布局和显示。



### 🏷️ Binder通信相关

1. Binder通信的基本流程，分为几个关键步骤
   1. **服务的注册与查找**：
      - **服务端**：首先，服务端通过`Service Manager`注册自己，以便客户端能够查找到它。通常，服务端会通过一个特定的名称将服务注册到`Service Manager`。
      - **客户端**：客户端通过`Service Manager`查找所需的服务，获取到对应服务的`Binder`引用。
   2. **请求的发送**：
      - 客户端通过Binder接口向服务端发送请求。请求通过`Parcel`对象进行封装，`Parcel`是一种用于打包数据的容器，可以在进程间传递数据。
      - 在发送请求时，客户端会调用`Binder`对象的`transact()`方法，这会将请求发送给内核空间的Binder驱动。
   3. **请求的处理**：
      - Binder驱动接收到客户端的请求后，将请求转发给目标服务端所在的进程。Binder驱动会解析`Parcel`对象，并通过共享内存将其传递给服务端。
      - 服务端接收到请求后，使用`Binder`接口实现具体的业务逻辑，然后将结果封装到`Parcel`中返回给客户端。
   4. **响应的接收**：
      - 客户端接收到服务端的响应，并通过`Parcel`对象解析返回的数据，从而完成一次Binder通信。

