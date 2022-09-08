---
title: RecyclerView缓存机制总结
date: 2019-08-15 00:31:17
tags:
- Android
- View
categories:
- Android
- View
---

# RecyclerView缓存机制总结

## 基本概念
**scrapped:** 
即 dettach 和 attach 。对一个view做dettach操作，会把这个view从 ViewGroup 的 children 数组里移除，但是不会重绘UI。 
在 LayoutManager 布局时，会 dettach 掉所有的子view，再从 Recycler 中获取view（缓存或新创建）后再 addView。 因为做过 dettach ，已经从 children 数组里移除了。所以再addView不会造成问题。 


## RecyclerView中涉及到缓存的集合

* mAttachedScrap
    * 显示在屏幕中，未与RecyclerView分离但被标记移除的Holder。 LayoutManager 布局时的临时缓存，布局完成后为空
* mChangedScrap
    * 显示在屏幕中，和 mAttachedScrap 类似，在预布局的时候会用到
* mCachedViews 
    * 在屏幕外的Holder。缓存，默认大小为2。
* mRecyclerPool
    * 在屏幕外的Holder。当mCachedViews满时，存储至此。按照ViewType进行分类存储。默认大小为5。从中取出的Holder需要调用onBindViewHolder方法

mCachedViews中取出的Holder是直接可用的，不需要调用onCreatedViewHolder和onBindViewHolder方法。

### mAttachedScrap 和 mChangedScrap 的插入
```java
void scrapView(View view) {
    final ViewHolder holder = getChildViewHolderInt(view);
    if (holder.hasAnyOfTheFlags(ViewHolder.FLAG_REMOVED | ViewHolder.FLAG_INVALID)
            || !holder.isUpdated() || canReuseUpdatedViewHolder(holder)) {
        if (holder.isInvalid() && !holder.isRemoved() && !mAdapter.hasStableIds()) {
            throw ..
        }
        holder.setScrapContainer(this, false);
        mAttachedScrap.add(holder);
    } else {
        if (mChangedScrap == null) {
            mChangedScrap = new ArrayList<ViewHolder>();
        }
        holder.setScrapContainer(this, true);
        mChangedScrap.add(holder);
    }
}
```
LayoutManager 在布局之前会scrap所有的view。 会根据不同的情况放入到 mAttachedScrap 或者 mChangedScrap
试了下，一般都是放在 mAttachedScrap 里，即使 holder.isUpdated() 为true。 所以 mAttachedScrap 里的ViewHolder也是有可能调用 onBind 的。
mChangedScrap 试了下只在有一些动画需要 preLayout 的时候有用到

> ViewHolder 只有在满足下面情况才会被添加到 mChangedScrap：当它关联的 item 发生了变化（notifyItemChanged 或者 notifyItemRangeChanged 被调用），并且 ItemAnimator 调用 ViewHolder#canReuseUpdatedViewHolder 方法时，返回了 false。否则，ViewHolder 会被添加到AttachedScrap 中。

> canReuseUpdatedViewHolder 返回 “false” 表示我们要执行用一个 view 替换另一个 view 的动画，例如淡入淡出动画。 “true”表示动画在 view 内部发生。

> mAttachedScrap 在 整个布局过程中都能使用，但是 changed scrap — 只能在预布局阶段使用。

## mCachedViews 和 mRecyclerPool 的插入
```java
void recycleViewHolderInternal(ViewHolder holder) {
    final boolean transientStatePreventsRecycling = holder
            .doesTransientStatePreventRecycling();
    @SuppressWarnings("unchecked") final boolean forceRecycle = mAdapter != null
            && transientStatePreventsRecycling
            && mAdapter.onFailedToRecycleView(holder);
    boolean cached = false;
    boolean recycled = false;
    if (forceRecycle || holder.isRecyclable()) {
        if (mViewCacheMax > 0
                && !holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID
                | ViewHolder.FLAG_REMOVED
                | ViewHolder.FLAG_UPDATE
                | ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN)) {
            // Retire oldest cached view
            int cachedViewSize = mCachedViews.size();
            if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {
                recycleCachedViewAt(0);
                cachedViewSize--;
            }

            int targetCacheIndex = cachedViewSize;
            if (ALLOW_THREAD_GAP_WORK
                    && cachedViewSize > 0
                    && !mPrefetchRegistry.lastPrefetchIncludedPosition(holder.mPosition)) {
                int cacheIndex = cachedViewSize - 1;
                while (cacheIndex >= 0) {
                    int cachedPos = mCachedViews.get(cacheIndex).mPosition;
                    if (!mPrefetchRegistry.lastPrefetchIncludedPosition(cachedPos)) {
                        break;
                    }
                    cacheIndex--;
                }
                targetCacheIndex = cacheIndex + 1;
            }
            mCachedViews.add(targetCacheIndex, holder);
            cached = true;
        }
        if (!cached) {
            addViewHolderToRecycledViewPool(holder, true);
            recycled = true;
        }
    }
    ...
}
void addViewHolderToRecycledViewPool(@NonNull ViewHolder holder, boolean dispatchRecycled) {
    ...
    holder.mBindingAdapter = null;
    holder.mOwnerRecyclerView = null; 
    getRecycledViewPool().putRecycledView(holder);
}
```
先尝试加入到 mCachedViews 集合，如果满了，就删除第一个
为加入到 mCachedViews 的会加入到 recyclerPool 里
mCacheViews 缓存是区分 viewType 的， recyclerPool 会区分存储，单个 viewType 容量是5
判定 `!holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID
                | ViewHolder.FLAG_REMOVED
                | ViewHolder.FLAG_UPDATE
                | ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN` 为true，才会加入到 mCachedViews 。 所以从 mCachedViews 获取是不需要重新 onBind 的
加入到 recyclerPool 时会设置 `holder.mBindingAdapter = null` 下面会说到这个


## RecyclerView获取Holder的顺序(sdk 28)
LayoutManager 在布局的时候会调用 `getViewForPosition(int position)` 方法获取 VH 和 View 
后续会调用到tryGetViewHolderForPositionByDeadline 中获取viewholder缓存，如果不存在会创建。

1. getChangedScrapViewForPosition
2. getScrapOrHiddenOrCachedHolderForPosition
3. getScrapOrCachedViewForId
4. getChildViewHolder
5. mViewCacheExtension.getViewForPositionAndType
6. getRecycledViewPool().getRecycledView
7. mAdapter.createViewHolder

从各种缓存集合中获取 ViewHolder 。 

## 获取的 ViewHolder 是否需要重新 bind
```java
boolean bound = false;
if (mState.isPreLayout() && holder.isBound()) {
    // do not update unless we absolutely have to.
    holder.mPreLayoutPosition = position;
} else if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
    final int offsetPosition = mAdapterHelper.findPositionOffset(position);
    bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);
}

private boolean tryBindViewHolderByDeadline(@NonNull ViewHolder holder, int offsetPosition,
       int position, long deadlineNs) {
   holder.mBindingAdapter = null;
   holder.mOwnerRecyclerView = RecyclerView.this;
   final int viewType = holder.getItemViewType();
   
   mAdapter.bindViewHolder(holder, offsetPosition);
   return true;
}

public final void bindViewHolder(@NonNull VH holder, int position) {
   boolean rootBind = holder.mBindingAdapter == null;
   if (rootBind) {
       holder.mPosition = position;
       if (hasStableIds()) {
           holder.mItemId = getItemId(position);
       }
       holder.setFlags(ViewHolder.FLAG_BOUND,
               ViewHolder.FLAG_BOUND | ViewHolder.FLAG_UPDATE | ViewHolder.FLAG_INVALID
                       | ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN);
       TraceCompat.beginSection(TRACE_BIND_VIEW_TAG);
   }
   holder.mBindingAdapter = this;
   onBindViewHolder(holder, position, holder.getUnmodifiedPayloads());
   ...
}
```

`!holder.isBound()` ViewHolder 是否调用过 onBind 。在bind时，如果 holder.mBindingAdapter == null ，会设置这个 bound 标记位。
`holder.needsUpdate()` 调用 notifyXxxxx 方法时vh会为true
`holder.isInvalid()` vh非法

* 是否调用 vh.onBind 只和标记位有关系，和从哪个缓存集合中获取到的无关。 
* 加入到 mCacheView 时会判断一定没有 needUpdate 和 Invalide 的标记位，所以mCacheView一定不需要重新bind
* recyclerPool 加入元素时设置了 `holder.mBindingAdapter = null` ，所以一定需要重新 bind
* mAttachedScrap 是否需要重新bind是不一定的

   

## ListView的缓存机制

### 缓存的集合
* mActiveViews 
    * 屏幕内的view，可直接重用
* mScrapViews
    * 屏幕外的view，需要调用bind

### 与RecyclerView的不同
1. 缓存不同： RecyclerView缓存的是ViewHolder，避免了每次的findViewByid，ListView缓存的是View。
2. RecyclerView中mCacheViews(屏幕外)获取缓存时，是通过匹配pos获取目标位置的缓存，这样做的好处是，当数据源数据不变的情况下，无须重新bindView。 
而同样是离屏缓存，ListView从mScrapViews根据pos获取相应的缓存，但是并没有直接使用，而是重新getView（即必定会重新bindView）
3. RecyclerView可以实现局部刷新， ListView不行


## 参考：
[RecyclerView源码分析缓存机制](https://blog.jiahuan.me/2018/07/27/RecyclerView%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E7%BC%93%E5%AD%98%E6%9C%BA%E5%88%B6/)

[RecyclerView的缓存机制](https://www.jianshu.com/p/efe81969f69d)

[Android ListView 与 RecyclerView 对比浅析--缓存机制](https://mp.weixin.qq.com/s?__biz=MzA3NTYzODYzMg==&mid=2653578065&idx=2&sn=25e64a8bb7b5934cf0ce2e49549a80d6&chksm=84b3b156b3c43840061c28869671da915a25cf3be54891f040a3532e1bb17f9d32e244b79e3f&scene=21#wechat_redirect)

[深入理解 RecyclerView 的缓存机制](https://codeantenna.com/a/3A2N3H6x8Y)