---
title: RequestLayout和Invalidate
date: 2022-05-02 16:48:49
tags:
---

# RequestLayout

## 标记自身

给当前View添加上 PFLAG_FORCE_LAYOUT 和 PFLAG_INVALIDATED 标记。
并将RequestLayout向上传递。

```java
//从源码注释可以看出，如果当前View在请求布局的时候，View树正在进行布局流程的话，
//该请求会延迟到布局流程完成后或者绘制流程完成且下一次布局发现的时候再执行。
@CallSuper
public void requestLayout() {
    if (mMeasureCache != null) mMeasureCache.clear();

    if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == null) {
        // Only trigger request-during-layout logic if this is the view requesting it,
        // not the views in its parent hierarchy
        ViewRootImpl viewRoot = getViewRootImpl();
        if (viewRoot != null && viewRoot.isInLayout()) {
            if (!viewRoot.requestLayoutDuringLayout(this)) {
                return;
            }
        }
        mAttachInfo.mViewRequestingLayout = this;
    }

    //为当前view设置标记位 PFLAG_FORCE_LAYOUT
    mPrivateFlags |= PFLAG_FORCE_LAYOUT;
    mPrivateFlags |= PFLAG_INVALIDATED;

    if (mParent != null && !mParent.isLayoutRequested()) {
        //向父容器请求布局
        mParent.requestLayout();
    }
    if (mAttachInfo != null && mAttachInfo.mViewRequestingLayout == this) {
        mAttachInfo.mViewRequestingLayout = null;
    }
}
```

## ViewRootImpl

会顺着view树，一路向上标记，最终到达 ViewRootImpl
```java
@Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals(); // 开启measure、layout、draw三步走
    }
}
```

 scheduleTraversals() 开启绘制三步走流程。 在measure、layout、draw过程中都会判断标志位，只有需要的时候才执行

 ```java
 public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
     ...

    if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||
            widthMeasureSpec != mOldWidthMeasureSpec ||
            heightMeasureSpec != mOldHeightMeasureSpec) {
        ...
        if (cacheIndex < 0 || sIgnoreMeasureCache) {
            // measure ourselves, this should set the measured dimension flag back
            onMeasure(widthMeasureSpec, heightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        } 
        ...
        mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
    }
}
 ```

 measure时判断了PFLAG_FORCE_LAYOUT标记，需要时才做。
  在 `mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;` 中给layout过程做了标记

 ```java
 public void layout(int l, int t, int r, int b) {
    ...
    //判断标记位是否为PFLAG_LAYOUT_REQUIRED，如果有，则对该View进行布局
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);
        //onLayout方法完成后，清除PFLAG_LAYOUT_REQUIRED标记位
        mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnLayoutChangeListeners != null) {
            ArrayList<OnLayoutChangeListener> listenersCopy =
                    (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
            int numListeners = listenersCopy.size();
            for (int i = 0; i < numListeners; ++i) {
                listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
            }
        }
    }

    //最后清除PFLAG_FORCE_LAYOUT标记位
    mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
    mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
}
 ```

 layout时判断了PFLAG_LAYOUT_REQUIRED标记，需要时才做。

 # Invalidate

 ## 标记自身
 ```java
 void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
        boolean fullInvalidate) {
    
    ...

    //根据View的标记位来判断该子View是否需要重绘，假如View没有任何变化，那么就不需要重绘
    if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
            || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
            || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
            || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
        ...

        //设置PFLAG_DIRTY标记位
        mPrivateFlags |= PFLAG_DIRTY;

        ...

        // Propagate the damage rectangle to the parent view.
        //把需要重绘的区域传递给父容器
        final AttachInfo ai = mAttachInfo;
        final ViewParent p = mParent;
        if (p != null && ai != null && l < r && t < b) {
            final Rect damage = ai.mTmpInvalRect;
            damage.set(l, t, r, b);
            //调用父容器的方法，向上传递事件
            p.invalidateChild(this, damage);
        }
        ...
    }
}
 ```

 ## ViewGroup处理并向上传递

 ```java
 /**
 * Don't call or override this method. It is used for the implementation of
 * the view hierarchy.
 */
public final void invalidateChild(View child, final Rect dirty) {

    //设置 parent 等于自身
    ViewParent parent = this;

    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        ...
        int opaqueFlag = isOpaque ? PFLAG_DIRTY_OPAQUE : PFLAG_DIRTY;

        if (child.mLayerType != LAYER_TYPE_NONE) {
            mPrivateFlags |= PFLAG_INVALIDATED;
            mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
        }
        
        ...

        do {
            View view = null;
            if (parent instanceof View) {
                view = (View) parent;
            }

            ...
            if (view != null) {
                if ((view.mViewFlags & FADING_EDGE_MASK) != 0 &&
                        view.getSolidColor() == 0) {
                    opaqueFlag = PFLAG_DIRTY;
                }
                if ((view.mPrivateFlags & PFLAG_DIRTY_MASK) != PFLAG_DIRTY) {
                    //对当前View的标记位进行设置
                    view.mPrivateFlags = (view.mPrivateFlags & ~PFLAG_DIRTY_MASK) | opaqueFlag;
                }
            }

            //调用ViewGrup的invalidateChildInParent，如果已经达到最顶层view,则调用ViewRootImpl的invalidateChildInParent。
            parent = parent.invalidateChildInParent(location, dirty);
            ...
        } while (parent != null);
    }
}

```

```java
public ViewParent invalidateChildInParent(final int[] location, final Rect dirty) {
    if ((mPrivateFlags & PFLAG_DRAWN) == PFLAG_DRAWN ||
            (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID) {
        if ((mGroupFlags & (FLAG_OPTIMIZE_INVALIDATE | FLAG_ANIMATION_DONE)) !=
                    FLAG_OPTIMIZE_INVALIDATE) {

            //将dirty中的坐标转化为父容器中的坐标，考虑mScrollX和mScrollY的影响
            dirty.offset(location[CHILD_LEFT_INDEX] - mScrollX,
                    location[CHILD_TOP_INDEX] - mScrollY);

            if ((mGroupFlags & FLAG_CLIP_CHILDREN) == 0) {
                //求并集，结果是把子视图的dirty区域转化为父容器的dirty区域
                dirty.union(0, 0, mRight - mLeft, mBottom - mTop);
            }

            final int left = mLeft;
            final int top = mTop;

            if ((mGroupFlags & FLAG_CLIP_CHILDREN) == FLAG_CLIP_CHILDREN) {
                if (!dirty.intersect(0, 0, mRight - left, mBottom - top)) {
                    dirty.setEmpty();
                }
            }
            mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;

            //记录当前视图的mLeft和mTop值，在下一次循环中会把当前值再向父容器的坐标转化
            location[CHILD_LEFT_INDEX] = left;
            location[CHILD_TOP_INDEX] = top;

            if (mLayerType != LAYER_TYPE_NONE) {
                mPrivateFlags |= PFLAG_INVALIDATED;
            }
            //返回当前视图的父容器
            return mParent;

        }
        ...
    }
    return null;
}
```

## ViewRootImpl
```java
@Override
public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
    checkThread();
    ...
    invalidateRectOnScreen(dirty);
    return null;
}

private void invalidateRectOnScreen(Rect dirty) {
    ...
    if (!mWillDrawSoon && (intersected || mIsAnimating)) {
        scheduleTraversals();
    }
}
```

```java
private boolean draw(boolean fullRedrawNeeded) {
    // 1. 使用硬件加速渲染
    // 2. 使用软件加速渲染
}

/**
     * @return true if drawing was successful, false if an error occurred
     */
    private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
            boolean scalingRequired, Rect dirty) {

        // Draw with software renderer.
        final Canvas canvas;
        try {
            final int left = dirty.left;
            final int top = dirty.top;
            final int right = dirty.right;
            final int bottom = dirty.bottom;

            canvas = mSurface.lockCanvas(dirty);

            // The dirty rectangle can be modified by Surface.lockCanvas()
            //noinspection ConstantConditions
            if (left != dirty.left || top != dirty.top || right != dirty.right
                    || bottom != dirty.bottom) {
                attachInfo.mIgnoreDirtyState = true;
            }

            // TODO: Do this in native
            canvas.setDensity(mDensity);
        } catch {
            ...
        }

        // 伪代码
        mView.mPrivateFlags |= View.PFLAG_DRAWN;
        mView.draw(canvas);
        
        return true;
    }
```
这里只有软件绘制的代码。 
网上有人说invalidate只会重绘脏区域，但是这里只看到用dirty做了一个 `canvas = mSurface.lockCanvas(dirty);` 。还不是很懂局部重绘是怎么做到的。。。。





