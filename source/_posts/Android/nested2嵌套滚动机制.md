---
title: nested2嵌套滚动机制
date: 2021-07-24 16:36:31
tags:
- nested2
---

## 开头
nested2是安卓用于处理嵌套滚动的方案。 主要有两个重要的接口， NestedScrollingParent2 和 NestedScrollingChild2 ，分别代表桥套滚动的外层和内层。

## NestedScrollingParent2

NestedScrollingParent2 包含以下接口

```java
public interface NestedScrollingParent2 extends NestedScrollingParent {
/**
    * 即将开始嵌套滑动，此时嵌套滑动尚未开始，由子控件的 startNestedScroll 方法调用
    *
    * @param child  嵌套滑动对应的父类的子类(因为嵌套滑动对于的父控件不一定是一级就能找到的，可能挑了两级父控件的父控件，child的辈分>=target)
    * @param target 具体嵌套滑动的那个子类
    * @param axes   嵌套滑动支持的滚动方向
    * @param type   嵌套滑动的类型，有两种ViewCompat.TYPE_NON_TOUCH fling效果,ViewCompat.TYPE_TOUCH 手势滑动
    * @return true 表示此父类开始接受嵌套滑动，只有true时候，才会执行下面的 onNestedScrollAccepted 等操作
    */
   boolean onStartNestedScroll(@NonNull View child, @NonNull View target, @ScrollAxis int axes,
           @NestedScrollType int type);

   /**
    * 当onStartNestedScroll返回为true时，也就是父控件接受嵌套滑动时，该方法才会调用
    *
    * @param child
    * @param target
    * @param axes
    * @param type
    */
   void onNestedScrollAccepted(@NonNull View child, @NonNull View target, @ScrollAxis int axes,
           @NestedScrollType int type);

   /**
    * 在子控件开始滑动之前，会先调用父控件的此方法，由父控件先消耗一部分滑动距离，并且将消耗的距离存在consumed中，传递给子控件
    * 在嵌套滑动的子View未滑动之前
    * ，判断父view是否优先与子view处理(也就是父view可以先消耗，然后给子view消耗）
    *
    * @param target   具体嵌套滑动的那个子类
    * @param dx       水平方向嵌套滑动的子View想要变化的距离
    * @param dy       垂直方向嵌套滑动的子View想要变化的距离 dy<0向下滑动 dy>0 向上滑动
    * @param consumed 这个参数要我们在实现这个函数的时候指定，回头告诉子View当前父View消耗的距离
    *                 consumed[0] 水平消耗的距离，consumed[1] 垂直消耗的距离 好让子view做出相应的调整
    * @param type     滑动类型，ViewCompat.TYPE_NON_TOUCH fling效果,ViewCompat.TYPE_TOUCH 手势滑动
    */
   void onNestedPreScroll(@NonNull View target, int dx, int dy, @NonNull int[] consumed,
           @NestedScrollType int type);
           
   /**
    * 在 onNestedPreScroll 中，父控件消耗一部分距离之后，剩余的再次给子控件，
    * 子控件消耗之后，如果还有剩余，则把剩余的再次还给父控件
    *
    * @param target       具体嵌套滑动的那个子类
    * @param dxConsumed   水平方向嵌套滑动的子控件滑动的距离(消耗的距离)
    * @param dyConsumed   垂直方向嵌套滑动的子控件滑动的距离(消耗的距离)
    * @param dxUnconsumed 水平方向嵌套滑动的子控件未滑动的距离(未消耗的距离)
    * @param dyUnconsumed 垂直方向嵌套滑动的子控件未滑动的距离(未消耗的距离)
    * @param type     滑动类型，ViewCompat.TYPE_NON_TOUCH fling效果,ViewCompat.TYPE_TOUCH 手势滑动
    */
   void onNestedScroll(@NonNull View target, int dxConsumed, int dyConsumed,
           int dxUnconsumed, int dyUnconsumed, @NestedScrollType int type);

    /**
    * 停止滑动
    *
    * @param target
    * @param type     滑动类型，ViewCompat.TYPE_NON_TOUCH fling效果,ViewCompat.TYPE_TOUCH 手势滑动
    */
 void onStopNestedScroll(@NonNull View target, @NestedScrollType int type);
}
```

## NestedScrollingChild2
```java
public interface NestedScrollingChild2 extends NestedScrollingChild {

   /**
    * 开始滑动前调用，在惯性滑动和触摸滑动前都会进行调用，此方法一般在 onInterceptTouchEvent或者onTouch中，通知父类方法开始滑动
    * 会调用父类方法的 onStartNestedScroll onNestedScrollAccepted 两个方法
    *
    * @param axes 滑动方向
    * @param type 开始滑动的类型 the type of input which cause this scroll event
    * @return 有父视图并且开始滑动，则返回true 实际上就是看parent的 onStartNestedScroll 方法
    */
   boolean startNestedScroll(@ScrollAxis int axes, @NestedScrollType int type);

  /**
    * 子控件停止滑动，例如手指抬起，惯性滑动结束
    *
    * @param type 停止滑动的类型 TYPE_TOUCH，TYPE_NON_TOUCH
    */
   void stopNestedScroll(@NestedScrollType int type);

    /**
    * 判断是否有父View 支持嵌套滑动
    */
   boolean hasNestedScrollingParent(@NestedScrollType int type);

 /**
    * 在dispatchNestedPreScroll 之后进行调用
    * 当滑动的距离父控件消耗后，父控件将剩余的距离再次交个子控件，
    * 子控件再次消耗部分距离后，又继续将剩余的距离分发给父控件,由父控件判断是否消耗剩下的距离。
    * 如果四个消耗的距离都是0，则表示没有神可以消耗的了，会直接返回false，否则会调用父控件的
    * onNestedScroll 方法，父控件继续消耗剩余的距离
    * 会调用父控件的
    *
    * @param dxConsumed     水平方向嵌套滑动的子控件滑动的距离(消耗的距离)    dx<0 向右滑动 dx>0 向左滑动 （保持和 RecycleView 一致）
    * @param dyConsumed     垂直方向嵌套滑动的子控件滑动的距离(消耗的距离)    dy<0 向下滑动 dy>0 向上滑动 （保持和 RecycleView 一致）
    * @param dxUnconsumed   水平方向嵌套滑动的子控件未滑动的距离(未消耗的距离)dx<0 向右滑动 dx>0 向左滑动 （保持和 RecycleView 一致）
    * @param dyUnconsumed   垂直方向嵌套滑动的子控件未滑动的距离(未消耗的距离)dy<0 向下滑动 dy>0 向上滑动 （保持和 RecycleView 一致）
    * @param offsetInWindow 子控件在当前window的偏移量
    * @return 如果返回true, 表示父控件又继续消耗了
    */
   boolean dispatchNestedScroll(int dxConsumed, int dyConsumed,
           int dxUnconsumed, int dyUnconsumed, @Nullable int[] offsetInWindow,
           @NestedScrollType int type);

   /**
    * 子控件在开始滑动前，通知父控件开始滑动，同时由父控件先消耗滑动时间
    * 在子View的onInterceptTouchEvent或者onTouch中，调用该方法通知父View滑动的距离
    * 最终会调用父view的 onNestedPreScroll 方法
    *
    * @param dx             水平方向嵌套滑动的子控件想要变化的距离 dx<0 向右滑动 dx>0 向左滑动 （保持和 RecycleView 一致）
    * @param dy             垂直方向嵌套滑动的子控件想要变化的距离 dy<0 向下滑动 dy>0 向上滑动 （保持和 RecycleView 一致）
    * @param consumed       父控件消耗的距离，父控件消耗完成之后，剩余的才会给子控件，子控件需要使用consumed来进行实际滑动距离的处理
    * @param offsetInWindow 子控件在当前window的偏移量
    * @param type           滑动类型，ViewCompat.TYPE_NON_TOUCH fling效果,ViewCompat.TYPE_TOUCH 手势滑动
    * @return true    表示父控件进行了滑动消耗，需要处理 consumed 的值，false表示父控件不对滑动距离进行消耗，可以不考虑consumed数据的处理，此时consumed中两个数据都应该为0
    */
   boolean dispatchNestedPreScroll(int dx, int dy, @Nullable int[] consumed,
           @Nullable int[] offsetInWindow, @NestedScrollType int type);
}
```

## nested2机制，滚动的传递

一般情况下，事件是从child的触摸事件开始的，

1. 首先调用 `child.startNestedScroll()` 方法，此方法内部通过 `NestedScrollingChildHelper` 调用并返回 `parent.onStartNestedScroll()` 方法的结果，为 true，说明parent接受了嵌套滑动，同时会调用 `parent.onNestedScrollAccepted()` 方法，此时开始嵌套滑动；

2. 在滑动事件中，child通过 `child.dispatchNestedPreScroll()` 方法分配滑动的距离，内部会先调用 `parent.onNestedPreScroll()` 方法，由parent先处理滑动距离。

3. parent消耗完成之后，再将剩余的距离传递给child，child拿到parent使用完成之后的距离之后，自己再处理剩余的距离。

4. 如果此时子控件还有未处理的距离，则将剩余的距离再次通过 `child.dispatchNestedScroll()` 方法调用 `parent.onNestedScroll()` 方法，将剩余的距离交个parent来进行处理

5. 滑动结束之后，调用 child.stopNestedScroll()通知parent滑动结束，至此，触摸滑动结束

触摸滑动结束之后，child会继续进行惯性滑动，惯性滑动可以通过 Scroller 实现，具体滑动可以自己来处理，在fling过程中，和触摸滑动调用流程一样，需要注意 type 参数的区分，用来通知parent两种不同的滑动流程

## 一个栗子

### 预期目标
自己实现一个嵌套滚动的 parent 和 child， 满足以下效果
1. parent 包含 top 和 content 两部分，可滚动
2. 当 top 未完全隐藏是，滚动 content ，是外层的 parent 滚动
3. 当 top 完全隐藏，触摸滚动 content ，content 自己滚动
4. 当触摸 content 滚动到顶部， top 未完全漏出， 则 parent 滚动，直至 top 完全露出

### xml布局

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical">

    <com.example.test2.nest2_test.CustomNestedParent
        android:id="@+id/nested_parent"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <LinearLayout
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:orientation="vertical">

            <androidx.core.widget.NestedScrollView
                android:layout_width="match_parent"
                android:layout_height="wrap_content">

                <LinearLayout
                    android:id="@+id/view_top"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:background="@color/colorAccent"
                    android:orientation="vertical">

                    <View
                        android:layout_width="match_parent"
                        android:layout_height="200dp" />
                </LinearLayout>
            </androidx.core.widget.NestedScrollView>

            <com.example.test2.nest2_test.CustomNestedChild
                android:id="@+id/view_list"
                android:layout_width="match_parent"
                android:layout_height="1500dp" />
        </LinearLayout>
    </com.example.test2.nest2_test.CustomNestedParent>
</LinearLayout>
```
这里模仿 ScrollView 嵌套 ScrollView 的场景，即外部的 CustomNestedParent 嵌套 内部的 CustomNestedChild ，并解决他们的之间的滑动冲突
topVie2 外面包裹了一层 NestedScrollView，是为了让它支持 nested 机制

```kotlin
class NestedTestActivity : AppCompatActivity() {

    private lateinit var nestedParent: CustomNestedParent
    private lateinit var listView: LinearLayout

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContentView(R.layout.activity_nested_test)

        nestedParent = findViewById(R.id.nested_parent)
        val topView: View = findViewById(R.id.view_top)
        listView = findViewById(R.id.view_list)

        nestedParent.init(topView, listView)

        addListItems()
    }

    private fun addListItems() {
      // 填充 child， 这里模拟 child 是一个 recyclerview
        for (i in 0 until 100) {
            val textView = TextView(this)
            textView.layoutParams = ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, 100)
            textView.text = "position $i"
            listView.addView(textView)
        }
    }
}
```

### CustomNestedParent
```kotlin
class CustomNestedParent @JvmOverloads constructor(
    context: Context?,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : LinearLayout(context, attrs, defStyleAttr), NestedScrollingParent2 {

    val TAG = "CustomNestedParent"

    private val mNestedScrollingParentHelper = NestedScrollingParentHelper(this)
    private lateinit var topView: View
    private lateinit var nestedChild: View
    private var childrenHeight = 0

    override fun onFinishInflate() {
        super.onFinishInflate()
    }

    fun init(topView: View, contentView: View) {
        this.topView = topView
        this.nestedChild = contentView
    }

    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
      // 模仿 NestedScrollView
        var height = MeasureSpec.getSize(heightMeasureSpec)
        var width = MeasureSpec.getSize(widthMeasureSpec)
        childrenHeight = 0
        for (i in 0 until childCount) {
            val child = getChildAt(i)
            measureChild(child, widthMeasureSpec, MeasureSpec.UNSPECIFIED)
            childrenHeight += child.measuredHeight
        }
        setMeasuredDimension(width, height)
    }

    // @return true 表示此父类开始接受嵌套滑动，只有true时候，才会执行其他操作
    override fun onStartNestedScroll(child: View, target: View, axes: Int, type: Int): Boolean {
        Log.i(TAG, "onStartNestedScroll: ")
        return axes == ViewCompat.SCROLL_AXIS_VERTICAL
    }

    // 当嵌套滑动被parent接收了，会回调这个方法
    override fun onNestedScrollAccepted(child: View, target: View, axes: Int, type: Int) {
        Log.i(TAG, "onNestedScrollAccepted: ")
        mNestedScrollingParentHelper.onNestedScrollAccepted(child, target, axes, type)
    }

    /**
     * 在子控件开始滑动之前，会先调用父控件的此方法，由父控件先消耗一部分滑动距离，并且将消耗的距离存在consumed中，传递给子控件
     * 不管手势滚动还是fling都会回调这个方法
     */
    override fun onNestedPreScroll(target: View, dx: Int, dy: Int, consumed: IntArray, type: Int) {
        val threshold = nestedChild.top
        var parentScrollable = false

        val hideTop = dy > 0 && scrollY < threshold
        val showTop = dy < 0 && !target.canScrollVertically(-1)
        Log.i(TAG, "onNestedPreScroll-1: $dy $type")
        if (hideTop || showTop) {
          // parent 提前消费的场景 
          // 1. 向上滚动，parent滚动的距离 < topView的高度，需要隐藏topView 
          // 2. 向下滚动，且 child 已经滚动到顶部，需要漏出 topView
            parentScrollable = true
            consumed[1] = dy
            scrollBy(0, dy)
            Log.i(TAG, "onNestedPreScroll-2: hideTop=$hideTop showTop=$showTop dy=$dy scrollY=$scrollY threshold=$threshold type=$type")
        } else {
            // 反之，应该让 child 滚动，parent不应该消费滚动距离
            parentScrollable = false
        }
    }

    /**
     * 在 onNestedPreScroll 中，父控件消耗一部分距离之后，剩余的再次给子控件，
     * 子控件消耗之后，如果还有剩余，则把剩余的再次还给父控件
     *
     * 孩子吃剩下的留给爸爸了
     */
    override fun onNestedScroll(
        target: View,
        dxConsumed: Int,
        dyConsumed: Int,
        dxUnconsumed: Int,
        dyUnconsumed: Int,
        type: Int
    ) {
        Log.i(TAG, "onNestedScroll: $dyUnconsumed $type $scrollY")
        // 剩余的parent全部消费
        scrollBy(0, dyUnconsumed)
    }

    override fun onStopNestedScroll(target: View, type: Int) {
        mNestedScrollingParentHelper.onStopNestedScroll(target, type);
    }

    override fun getNestedScrollAxes(): Int {
        return mNestedScrollingParentHelper.nestedScrollAxes
    }

    override fun scrollTo(x: Int, y: Int) {
        var resY = y
        // 限定 parnet 的上下边界，防止滚动出屏幕外
        if (resY < 0) {
            resY = 0
        }
        val max = max(childrenHeight - height, 0)
        if (y > max) {
            resY = max
        }
        Log.i(TAG, "scrollTo: $max $y $resY")
        super.scrollTo(x, resY)
    }
}
```

### CustomNestedChild

```kotlin
class CustomNestedChild @JvmOverloads constructor(
    context: Context?,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : LinearLayout(context, attrs, defStyleAttr), NestedScrollingChild2 {

    val TAG = "CustomNestedChild"

    private val mScrollingChildHelper = NestedScrollingChildHelper(this)

    private val viewConfiguration: ViewConfiguration = ViewConfiguration.get(context)
    private var mVelocityTracker: VelocityTracker? = null

    private val mScroller: Scroller = Scroller(context)

    private var mLastX: Float = 0f
    private var mLastY: Float = 0f
    private var mLastFlingX: Float = 0f
    private var mLastFlingY: Float = 0f

    private val offset = IntArray(2)
    private val consumed = IntArray(2)
    private var fling = false //判断当前是否是可以进行惯性滑动

    private var childrenHeight = 0

    init {
        orientation = VERTICAL
        // 这里必须都设置为 true ，表明这个view是支持nested2机制的
        isNestedScrollingEnabled = true
        mScrollingChildHelper.isNestedScrollingEnabled = true
    }

    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        var height = MeasureSpec.getSize(heightMeasureSpec)
        var width = MeasureSpec.getSize(widthMeasureSpec)
        childrenHeight = 0
        for (i in 0 until childCount) {
            val child = getChildAt(i)
            measureChild(child, widthMeasureSpec, MeasureSpec.UNSPECIFIED)
            childrenHeight += child.measuredHeight
        }
        setMeasuredDimension(width, height)
    }

    override fun startNestedScroll(axes: Int, type: Int): Boolean {
        return mScrollingChildHelper.startNestedScroll(axes, type)
    }

    override fun stopNestedScroll(type: Int) {
        mScrollingChildHelper.stopNestedScroll(type)
    }

    override fun hasNestedScrollingParent(type: Int): Boolean {
        return mScrollingChildHelper.hasNestedScrollingParent()
    }

    override fun dispatchNestedPreScroll(
        dx: Int,
        dy: Int,
        consumed: IntArray?,
        offsetInWindow: IntArray?,
        type: Int
    ): Boolean {
        return mScrollingChildHelper.dispatchNestedPreScroll(dx, dy, consumed, offsetInWindow, type)
    }

    override fun dispatchNestedScroll(
        dxConsumed: Int,
        dyConsumed: Int,
        dxUnconsumed: Int,
        dyUnconsumed: Int,
        offsetInWindow: IntArray?,
        type: Int
    ): Boolean {
        return mScrollingChildHelper.dispatchNestedScroll(
            dxConsumed,
            dyConsumed,
            dxUnconsumed,
            dyUnconsumed,
            offsetInWindow,
            type
        )
    }

    override fun onTouchEvent(event: MotionEvent): Boolean {
        // 处理触摸事件是，关闭fling
        cancelFling()

        if (mVelocityTracker == null) {
            mVelocityTracker = VelocityTracker.obtain()
        }
        val velocityTracker = mVelocityTracker!!
        velocityTracker.addMovement(event)

        when (event.action) {
            MotionEvent.ACTION_DOWN -> {
                mLastX = event.x
                mLastY = event.y
                // 表明开始处理嵌套滚动， TYPE_TOUCH 表明这是一个触摸场景
                startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL, TYPE_TOUCH)
            }
            MotionEvent.ACTION_MOVE -> {
                val curX = event.x
                val curY = event.y
                var dy = (mLastY - curY).toInt()
                var dx = (mLastX - curX).toInt()
                // 先交给parent处理
                if (dispatchNestedPreScroll(dx, dy, consumed, offset, TYPE_TOUCH)) {
                    dy -= consumed[1]
                    dx -= consumed[0]
                }
                // child自己消费
                val consumedY = childConsumeY(dy)
                // 将消费剩下的，传递给parent
                dispatchNestedScroll(0, consumedY, dx, dy - consumedY, null, TYPE_TOUCH)
                mLastX = curX
                mLastY = curY
            }
            MotionEvent.ACTION_UP,
            MotionEvent.ACTION_CANCEL -> {
              // 先结束 TYPE_TOUCH 场景的嵌套滚动
                stopNestedScroll(TYPE_TOUCH)

                // 判断是否需要惯性滑动
                velocityTracker.computeCurrentVelocity(
                    1000,
                    viewConfiguration.scaledMaximumFlingVelocity.toFloat()
                )
                val yvel = velocityTracker.yVelocity
                fling(yvel.toInt())
                velocityTracker.clear()
            }
        }
        return true
    }

    private fun fling(velocityY: Int) {
        cancelFling()

        //判断速度是否足够大。如果够大才执行fling
        var dy: Int = velocityY
        if (abs(velocityY) < viewConfiguration.scaledMinimumFlingVelocity) {
            dy = 0
        }
        if (dy == 0) {
            return
        }

        // 这步必须要有，表明开始 TYPE_NON_TOUCH 的嵌套滚动
        // 如果不设置，后续的嵌套滚动流程都都会找不到对应的parent
        startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL, TYPE_NON_TOUCH)

        val maxFlingVelocity: Int = viewConfiguration.scaledMaximumFlingVelocity
        dy = max(-maxFlingVelocity, min(dy, maxFlingVelocity))

        Log.i(TAG, "fling: $dy ")
        fling = true
        // 开始fling
        mScroller.fling(
            0,
            0,
            0,
            dy,
            Integer.MIN_VALUE,
            Integer.MAX_VALUE,
            Integer.MIN_VALUE,
            Integer.MAX_VALUE
        )
        postInvalidate()
    }

    override fun computeScroll() {
        if (mScroller.computeScrollOffset() && fling) {
            val y = mScroller.currY
            var dy = (mLastFlingY - y).toInt()
            mLastFlingY = y.toFloat()
            // 和触摸场景一样，优先让parent处理
            if (dispatchNestedPreScroll(0, dy, consumed, null, TYPE_NON_TOUCH)) {
                dy -= consumed[1]
            }
            Log.i(TAG, "computeScroll: ${consumed[1]} $dy")
            // child 自己处理
            val consumedY = childFling(dy)
            // 将剩下的在传递给 parent
            dispatchNestedScroll(0, consumedY, 0, dy - consumedY, null, TYPE_NON_TOUCH)
            postInvalidate()
        } else {
            stopNestedScroll(TYPE_NON_TOUCH)
            cancelFling()
        }
    }

    private fun childConsumeY(dy: Int): Int {
        var consumed = dy
        if (consumed < 0) {
            // 如果滑动到顶部，只处理部分，不然可能无法触达顶部边界
            consumed = max(-scrollY, consumed)
        } else {
            // 如果滑动到顶部，只处理部分，不然可能会无法触达底部边界
            val max = max(childrenHeight - height, 0)
            if (dy + scrollY > max) {
                consumed = max - scrollY
            }
        }
        Log.i(TAG, "childConsumeY: $dy $consumed $scrollY")
        scrollBy(0, consumed)
        return consumed
    }

    private fun childFling(dy: Int): Int {
        return childConsumeY(dy)
    }

    private fun cancelFling() {
        fling = false
        mLastFlingY = 0f
        mLastFlingY = 0f
    }

    override fun scrollTo(x: Int, y: Int) {
        var resY = y
        if (resY < 0) {
            resY = 0
        }
        val max = max(childrenHeight - height, 0)
        if (y > max) {
            resY = max
        }
        Log.i(TAG, "scrollTo: $max $y $resY")
        super.scrollTo(x, resY)
    }

    override fun canScrollVertically(direction: Int): Boolean {
        if (direction < 0 && scrollY <= 0) {
            return false
        } else if (direction > 0 && scrollY >= measuredHeight - height) {
            return false
        } else {
            return true
        }
    }
}
```




