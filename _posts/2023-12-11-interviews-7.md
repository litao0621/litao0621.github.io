---
title: '面试相关（7）'
author: litao
date: 2023-12-11 18:34:00 +0800
categories: [interviews]
tags: [interviews]




---

### 🏷️  ViewModel 什么时候销毁

正常是跟随着LifecyclwOwner的生命周期（onDestroy）销毁的,但需要主要内部是有一个判断条件的，如果是发送了配置修改例如旋转屏幕，修改字体等一些操作，内部就会判断isChangingConfigurations，如果是配置修改是不会进行ViewModelStore等清除掉。











