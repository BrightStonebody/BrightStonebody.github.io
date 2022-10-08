---
title: Coordinatorlayout.Behavior使用
date: 2022-10-08 18:17:53
tags:
- Coordinatorlayout
---

## Behavior 的测量和布局

Behavior有 onMeasureChild , onLayoutChild 。 Coordinatorlayout 在执行 onMeasureChild 和 onLayoutChild 之前会先判断 behavior 是否有重写对应的方法并返回true。
如果返回true，则表示 behavior 接管了这个child的测量或布局，跳过该child。

## Behavior 的普通触摸事件

onInterceptTouchEvent , onTouchEvent 。 和 onMeasureChild , onLayoutChild 类似。

## Behavior 实现嵌套滚动机制

和正常的 NestedScroll 机制基本一致，api一一对应。

## Behavior 的 layoutDependsOn + onDependentViewChanged + onDependentViewRemoved


```java
    /**
     * child是否要依赖dependency
     * @param parent     CoordinatorLayout
     * @param child      该Behavior对应的那个View
     * @param dependency 要检查的View(child是否要依赖这个dependency)
     * @return true 依赖, false 不依赖
     */
    public boolean layoutDependsOn(CoordinatorLayout parent, V child, View dependency) {
        return false;
    }

    /**
     * 在layoutDependsOn返回true的基础上之后，及时报告dependency的状态变化
     * @param parent     CoordinatorLayout
     * @param child      该Behavior对应的那个View
     * @param dependency child依赖dependency
     * @return true 处理了, false  没处理
     */
    public boolean onDependentViewChanged(CoordinatorLayout parent, V child, View dependency) {
        return false;
    }

    /**
     * 在layoutDependsOn返回true的基础上之后，报告dependency被移除了
     * @param parent     CoordinatorLayout
     * @param child      该Behavior对应的那个View
     * @param dependency child依赖dependency
     */
    public void onDependentViewRemoved(CoordinatorLayout parent, V child, View dependency) {
    }
```
