---
title: 'H5页面内部div滑动与webview下拉刷新的冲突处理'
author: 李涛
date: 2022-05-04 11:10:00 +0900
categories: [Android,WebView]
tags: [WebView]
---

> 首先描述下问题 - 在H5页面中我们集成下拉刷新控件，无论是官网提供的`SwipeRefreshLayout`还是其他三方的下拉刷新库，通常都会去在`onScrollChanged`中去监听滑动状态，或使用`View.canScrollVertically()`去判断到顶到底等状态来控制下拉刷新的触发时机，但如果H5页面不是单纯的页面滚动，而是在内部div中添加了类似于`overflow-y: scroll`的滚动属性，我们将无法再获取到相应的滑动状态，从而导致下拉刷新无法正常工作。

## 解决方案

*简单说下目前主流的一些解决方案*

### 一. 禁用下拉刷新
控制我们页面在存在内部滚动形式的页面去禁用下拉刷新，但这样的方式在我们自己的页面中还可以去添加标识去控制，三方页面就很难处理了，也可以像很多其他应用一样整体把刷新控制修改为手动触发，添加到右上角的更多操作功能区中。

### 二. 通过js注入
通过js注入来拿到指定滑动元素的偏移量，从而判读出是否处于顶部来变更下拉刷新控件的状态

```kotlin
val js = "document.getElementById("scroll_view").scrollTop"
evaluateJavascript(js) {
    val scrollY = it.toInt()
    refreshView.setRefreshEnable(scrollY == 0)
}
```
这种方式也存在同样的问题，无法统一处理，必须先知道内部滑动容器的id或类名

### 三. 监听over scroll的状态
这种也是目前网络上提供的较多的一种方式，同时也去逆向了知乎的解决方案，也是通过此方式去实现的
``` kotlin
override fun onOverScrolled(scrollX: Int, scrollY: Int, clampedX: Boolean, clampedY: Boolean) {
    
    //我们可以通过clampedY来判断是否已经触发了纵向的over scroll
    val canRefresh = clampedY
    
    super.onOverScrolled(scrollX, scrollY, clampedX, clampedY)
}
```
但知乎的webview中同样也存在一个问题，就是首次进入和从新回到顶部后，都无法立即触发over scroll可能需要用户多次下拉才可以触发
### 四. 综合以上存在的问题进行整合处理
我们可以通过`onOverScrolled`状态来校验是否到达了边界，通过`onTouchEvent`来控制是否可刷新与事件传递的连贯性，可以通过`overScrollBy`方法来判断over scroll状态的方向（如果不判断我们滑动到页面底部也将出发over scroll导致我们无法正确启用下拉刷新），综合以上参数我们就可以准确的进行下拉刷新状态的控制

下面看下完整代码


``` kotlin
import android.content.Context
import android.util.AttributeSet
import android.view.MotionEvent
import android.webkit.WebView

/**
 * @author : litao
 * @email : onresume@live.com
 * @date : 2022/5/24 11:16 上午
 */
internal class TestWebView constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0,
) : WebView(context, attrs, defStyleAttr) {


    private var overScrolledY = false
    private var startY  = 0f
    private var refreshEnable = false
    private var isPullDown = false

    override fun onOverScrolled(scrollX: Int, scrollY: Int, clampedX: Boolean, clampedY: Boolean) {
        super.onOverScrolled(scrollX, scrollY, clampedX, clampedY)
        if (clampedY){
            overScrolledY = true
        }
    }

    override fun overScrollBy(
        deltaX: Int,
        deltaY: Int,
        scrollX: Int,
        scrollY: Int,
        scrollRangeX: Int,
        scrollRangeY: Int,
        maxOverScrollX: Int,
        maxOverScrollY: Int,
        isTouchEvent: Boolean
    ): Boolean {
        //判断当前over scroll 方向
        isPullDown = deltaY < 0

        return super.overScrollBy(deltaX,
            deltaY,
            scrollX,
            scrollY,
            scrollRangeX,
            scrollRangeY,
            maxOverScrollX,
            maxOverScrollY,
            isTouchEvent)
    }


    override fun onTouchEvent(event: MotionEvent?): Boolean {

        when (event?.actionMasked) {
            MotionEvent.ACTION_DOWN -> {
                startY = event.y
            }
            MotionEvent.ACTION_MOVE -> {
                val diffY = event.y - startY
                if (diffY < 0){
                    overScrolledY = false
                    refreshEnable = false
                    setRefreshEnable(false)
                }

                if (overScrolledY && !refreshEnable && isPullDown) {
                    refreshEnable = true
                    setRefreshEnable(true)
                    val obtain = MotionEvent.obtain(event)
                    obtain.action = MotionEvent.ACTION_DOWN
                    dispatchTouchEvent(obtain)
                    (parent as TDRefreshLayout).dispatchTouchEvent(obtain)
                }
            }
        }
        return super.onTouchEvent(event)
    }

    fun setRefreshEnable(enable : Boolean){
        // TODO: you refresh layout enable code
    }

}
```

Webview没有为我们提供明确的判断方法，经过以上处理，我们也可以达到一个近似系统浏览器的效果，如有更好的方案欢迎大家提供。