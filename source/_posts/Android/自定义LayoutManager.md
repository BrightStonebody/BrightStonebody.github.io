---
title: 自定义LayoutManager
date: 2021-02-26 17:06:00
tags:
- RecyclerView
- Android
---

# LayoutManager 的常用方法

### generateDefaultLayoutParams

生成默认RecyclerView的LayoutParams, 没啥意义，就返回 wrap_content

### isAutoMeasureEnabled 和 onMeasure

isAutoMeasureEnabled()是自测量模式。 如果要支持 wrap_content ，那必须重写这两个方法中的一个。 大部分情况下，isAutoMeasureEnabled 返回 true 即可。

### canScrollHorizontally 和 canScrollVertically 

无需多言

### onLayoutChildren 

当 RecyclerView 的 layout 过程中会调用这个方法，**包括第一次 layout 和 其他原因的重新 layout**，比如说键盘的升降。

### scrollHorizontallyBy 和 scrollVerticallyBy

```
override fun scrollHorizontallyBy(

    dx: Int,
    recycler: RecyclerView.Recycler,
    state: RecyclerView.State
): Int {
    ...
}
```

RecyclerView 滚动的时候回回调这个方法，但是它只是告诉你用户滑动的距离，需要你自己去实现滑动的效果。同时需要去判断滑动的边界。
但是返回值与 RecyclerView 的 overScorll 的动画有关，返回0会有边界的波纹动画。

### getPosition(View)

返回 child view 在 adapter 中的位置

### getDecoratedXxxx

在 LayoutManager 中与布局有关的api都需要替换成带 Decorated 的api，不然无法实现对 itemDecorated 的兼容

### detachAndScrapAttachedViews()

从 RecyclerView 暂时移除 child 并送入临时缓存。 之前一直很奇怪，暂时移除是个什么场景。 后来想明白了，在每次布局 child 之前，都需要调用一次这个方法，把6所有的 child 移除到临时缓存里。否则，在布局的时候 addView(child) ，如果 child 已经 add 到了 RecyclerView 里，会造成一些麻烦。

### removeAndRecycleView

移除 child 并将对应的 ViewHolder 移动到 cachePool ，从 cachePool 中取出的 ViewHolder 需要重新调用 Adapter.onBindViewHolder 方法

# 自定义 LayoutManager 的一般套路

scrollHorizontallyBy 和 onLayoutChildren 可以服用同一个布局逻辑。

1. 确定锚点 view 的 position
2. 确定布局的左右(上下)边界
3. 开始 addView()->measureView()->layoutView() 
4. 回收布局边界之外的 child

# demo代码

```kotlin
class CustomLinearLayoutManager : RecyclerView.LayoutManager() {

    override fun generateDefaultLayoutParams(): RecyclerView.LayoutParams {
        return RecyclerView.LayoutParams(
            ViewGroup.LayoutParams.WRAP_CONTENT,
            ViewGroup.LayoutParams.WRAP_CONTENT
        )
    }

    override fun isAutoMeasureEnabled(): Boolean {
        return true
    }

    override fun canScrollHorizontally(): Boolean {
        return true
    }

    override fun canScrollVertically(): Boolean {
        return false
    }

    override fun onLayoutChildren(recycler: RecyclerView.Recycler, state: RecyclerView.State) {
        if (state.itemCount == 0) {
            removeAndRecycleAllViews(recycler)
            return
        }

        var left = paddingStart
        var curPos = 0
        if (childCount > 0) {
            // 这里是兼容键盘升起然后重新 rebuild 的情况。
            // 如果不做兼容，RecyclerView 会移动到列表最顶部
            left = getDecoratedLeft(getChildAt(0)!!)
            curPos = getPosition(getChildAt(0)!!)
        }

        detachAndScrapAttachedViews(recycler)

        fill(recycler, state, curPos, left, paddingStart + getAvailableSpace(), true)
    }

    override fun scrollHorizontallyBy(
        dx: Int,
        recycler: RecyclerView.Recycler,
        state: RecyclerView.State
    ): Int {
        val resDx = calculateOffset(recycler, state, dx)
        if (resDx == 0 || itemCount == 0) {
            return resDx
        }

        val d = abs(resDx)
        if (resDx >= 0) {
            val anchorView = getChildAt(0)
            val anchorPos = getPosition(anchorView!!)
            val anchorLeft = getDecoratedLeft(anchorView)

            detachAndScrapAttachedViews(recycler)

            fill(recycler, state, anchorPos, anchorLeft - d, getAvailableSpace() + paddingStart, true)
        } else {
            val anchorView = getChildAt(childCount - 1)
            val anchorPos = getPosition(anchorView!!)
            val anchorRight = getDecoratedRight(anchorView)

            detachAndScrapAttachedViews(recycler)

            fill(recycler, state, anchorPos, paddingLeft, anchorRight + d, false)
        }

        recycler(recycler, state, resDx)

        return resDx
    }
    
    private fun calculateOffset(recycler: RecyclerView.Recycler, state: RecyclerView.State, dx: Int): Int {
        if (childCount == 0 || dx == 0) {
            return 0
        }

        var fillPos = RecyclerView.NO_POSITION
        val d = abs(dx)


        if (dx < 0) {
            val firstView = getChildAt(0)
            val firstPos = getPosition(firstView!!)
            val firstLeft = getDecoratedLeft(firstView)

            fillPos = firstPos - 1

            if (fillPos < 0 && firstLeft + d > paddingStart) {
                return firstLeft - paddingStart
            }
            if (firstLeft + d < paddingStart) {
                return dx
            }

        } else {
            val lastView = getChildAt(childCount - 1)
            val lastPos = getPosition(lastView!!)
            val lastRight = getDecoratedRight(lastView)

            fillPos = lastPos + 1
            val endEdge = getAvailableSpace() + paddingStart

            if (fillPos >= itemCount && lastRight - d < endEdge) {
                return lastRight - endEdge
            }
            if (lastRight - d > endEdge) {
                return dx
            }
        }
        
        return dx
    }

    private fun fill(recycler: RecyclerView.Recycler, state: RecyclerView.State,
                     anchorIndex: Int, anchorLeft: Int, anchorRight: Int, isLTR: Boolean) {
        var availableSpace = anchorRight - anchorLeft
        var fillPos = anchorIndex
        var left = anchorLeft
        var right = anchorRight
        val top = paddingTop

        while (availableSpace > 0 && fillPos >= 0 && fillPos < state.itemCount) {
            val view = recycler.getViewForPosition(fillPos)
            if (isLTR) {
                addView(view)
                measureChildWithMargins(view, 0, 0)
                right = left + getDecoratedMeasuredWidth(view)
                val bottom = top + getDecoratedMeasuredHeight(view)
                layoutDecoratedWithMargins(view, left, top, right, bottom)
                fillPos++
                left = right
            } else {
                addView(view, 0)
                measureChildWithMargins(view, 0, 0)
                left = right - getDecoratedMeasuredWidth(view)
                val bottom = top + getDecoratedMeasuredHeight(view)
                layoutDecoratedWithMargins(view, left, top, right, bottom)
                fillPos--
                right = left
            }

            availableSpace -= getDecoratedMeasuredWidth(view)
        }
    }

    private fun recycler(recycler: RecyclerView.Recycler, state: RecyclerView.State, dx: Int) {
        //要回收View的集合，暂存
        val recycleViews = hashSetOf<View>()

        //dx>0就是手指从右滑向左，所以要回收前面的children
        if (dx > 0) {
            for (i in 0 until childCount) {
                val child = getChildAt(i)!!
                val right = getDecoratedRight(child)
                //itemView的right<0就是要超出屏幕要回收View
                if (right > paddingStart) break
                recycleViews.add(child)
            }
        }
 
        //dx<0就是手指从左滑向右，所以要回收后面的children
        if (dx < 0) {
            for (i in childCount - 1 downTo 0) {
                val child = getChildAt(i)!!
                val left = getDecoratedLeft(child)

                //itemView的left>recyclerView.width就是要超出屏幕要回收View
                if (left < getAvailableSpace() + paddingStart) break
                recycleViews.add(child)
            }
        }

        //真正把View移除掉
        for (view in recycleViews) {
            removeAndRecycleView(view, recycler)
        }
    }

    private fun getAvailableSpace(): Int {
        return width - paddingStart - paddingEnd
    }
}
```