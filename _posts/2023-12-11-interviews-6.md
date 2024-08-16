---
title: '面试相关（6）'
author: litao
date: 2023-12-11 18:34:00 +0800
categories: [interviews]
tags: [interviews]



---

### 🏷️  Android 差值器、估值器

- 插值器：根据时间（动画时常）流逝的百分比来计算属性变化的百分比。系统默认的有匀速，加减速，减速插值器。
- 估值器：通过上面插值器得到的百分比计算出具体变化的值。系统默认的有整型，浮点型，颜色估值器

### 🏷️  ApplicationContext 和 ActivityContext的区别

1. 生命周期不同，ApplicationContext是与整个应用程序的生命周期绑定，直到应用程序终止时才被销毁。ActivityContext与`Activity`的生命周期绑定，当`Activity`创建时创建，`Activity`销毁时销毁。
2. 内存泄漏问题，ApplicationContext与应用生命周期绑定，不会产生内存泄漏，ActivityContext与Activity绑定，在异步操作中需要注意内存泄漏问题。
3. 通常与页面ui有交互的需要使用ActivityContext。无交互的如访问网络，数据库，系统服务等可以使用ApplicationContext











