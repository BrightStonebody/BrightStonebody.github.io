---
title: 'DecorView, Window, ViewRootImpl的关系'
date: 2022-08-19 10:22:30
tags:
---

## Window的创建

一个 activity 对应的是 PhoneWindow 。 创建的时机是在 attach 

```java
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
    attachBaseContext(context);

    mFragments.attachHost(null /*parent*/);

    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    
    ...

    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
    mWindowManager = mWindow.getWindowManager();
    ...
}
```

实例化 PhoneWindow 的同时，会设置 WindowManager , WindowManager 是一个系统服务。 在app进程的代理类是 WindowManagerImpl， 但是 WindowManagerImpl 把实现委托给了 WindowManagerGlobal 。 

## DecorView 的创建
DecorView的创建在 `activity.setContentView(int)` , 最终实现在 `PhoneWindow.setContentView(int)`

```java
@Override
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    ...
    // mContentParent 是 android.R.id.content 的 view
    mLayoutInflater.inflate(layoutResID, mContentParent);
    ...
}

private void installDecor() {
    mForceDecorInstall = false;
    if (mDecor == null) {
        mDecor = generateDecor(-1); // 实例化 DecorView 
        ...
    } else {
        mDecor.setWindow(this);
    }
    if (mContentParent == null) {
        mContentParent = generateLayout(mDecor); // 返回 content 的view
    }
}
```

## ViewRootImpl 的创建

ViewRootImpl 的创建在 `ActivityThread#handleResumeActivity` 里。
```java
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
        String reason) {
    
    // 调用 activity#onResume
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    ...

    final Activity a = r.activity;
    final int forwardBit = isForward
            ? WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;

    // If the window hasn't yet been added to the window manager,
    // and this guy didn't finish itself or start another activity,
    // then go ahead and add the window.
    boolean willBeVisible = !a.mStartedActivity;
    if (!willBeVisible) {
        try {
            willBeVisible = ActivityTaskManager.getService().willActivityBeVisible(
                    a.getActivityToken());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
    if (r.window == null && !a.mFinished && willBeVisible) {
        r.window = r.activity.getWindow();
        View decor = r.window.getDecorView();
        decor.setVisibility(View.INVISIBLE);
        ViewManager wm = a.getWindowManager();
        WindowManager.LayoutParams l = r.window.getAttributes();
        a.mDecor = decor;
        ...
        if (a.mVisibleFromClient) {
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
                wm.addView(decor, l); // 主流程下一步
            } else {
                a.onWindowAttributesChanged(l);
            }
        }
    } else if (!willBeVisible) {
        r.hideForNow = true;
    }
    ...

    // The window is now visible if it has been added, we are not
    // simply finishing, and we are not starting another activity.
    if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
        ViewRootImpl impl = r.window.getDecorView().getViewRootImpl();
        WindowManager.LayoutParams l = impl != null
                ? impl.mWindowAttributes : r.window.getAttributes();
        ...
        r.activity.mVisibleFromServer = true;
        mNumVisibleActivities++;
        if (r.activity.mVisibleFromClient) {
            r.activity.makeVisible();
        }
    }
    
    Looper.myQueue().addIdleHandler(new Idler());
}
```

调用了 WindowManagerGlobal#addView(...)

activity#makeVisible()
```java
void makeVisible() {
    ...
    mDecor.setVisibility(View.VISIBLE);
}
```

```java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow, int userId) {
    ...

    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;

    ViewRootImpl root; // 实例化 ViewRootImpl
    View panelParentView = null;

    synchronized (mLock) {
        int index = findViewLocked(view, false);
        if (index >= 0) {
            if (mDyingViews.contains(view)) {
                mRoots.get(index).doDie();
            } else {
                throw 
            }
            // The previous removeView() had not completed executing. Now it has.
        }

        root = new ViewRootImpl(view.getContext(), display);

        view.setLayoutParams(wparams);

        // 存储所有的 decorView 和 ViewRootImpl 。暂时不明白存储来做什么的
        mViews.add(view); 
        mRoots.add(root);
        mParams.add(wparams);

        try {
            // 绑定 DecorView 和 ViewRootImpl
            root.setView(view, wparams, panelParentView, userId);
        } catch (RuntimeException e) {
            if (index >= 0) {
                // 异常了，移除
                removeViewLocked(index, true);
            }
            throw e;
        }
    }
}
```

### ViewRootImpl 

核心类是 ViewRootImpl 。 它负责和系统 WMS 交互

```java
public final class ViewRootImpl implements ViewParent,
        View.AttachInfo.Callbacks, ThreadedRenderer.DrawCallbacks {
    final ViewRootHandler mHandler = new ViewRootHandler();

    public ViewRootImpl(Context context, Display display) {
        mContext = context;
        // WMS端Session的代理对象
        mWindowSession = WindowManagerGlobal.getWindowSession();
        mDisplay = display;
        mThread = Thread.currentThread();
        // 继承于IWindow.Stub的W对象
        mWindow = new W(this);
        mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this, context);
        // 绘制相关的对象
        mChoreographer = Choreographer.getInstance();
        // ...
    }

    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;
                mAttachInfo.mRootView = view;
                // ...
            }
            // 开启绘制流程三步走
            requestLayout();
            // 由之前的解析可知Session.addToDisplay会调用到WMS.addWindow方法
            // 通过session与WMS建立通信，完成window的添加
            // mWindowSession app进程只会有一个
            // mWindow 用于binder双向通信，ViewRootImpl 都会有一个
            es = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                    getHostVisibility(), mDisplay.getDisplayId(), mWinFrame,
                    mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                    mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel);
            // ...
        }
    }
}
```

## Window 的移除

在 ActivityThread 的 handleDestroyActivity 中

```java
@Override
public void handleDestroyActivity(IBinder token, boolean finishing, int configChanges,
        boolean getNonConfigInstance, String reason) {
    // 调用 activity#onDestroy
    ActivityClientRecord r = performDestroyActivity(token, finishing,
            configChanges, getNonConfigInstance, reason);
    if (r != null) {
        WindowManager wm = r.activity.getWindowManager();
        View v = r.activity.mDecor;
        if (v != null) {
            if (r.activity.mVisibleFromServer) {
                mNumVisibleActivities--;
            }
            IBinder wtoken = v.getWindowToken();
            if (r.activity.mWindowAdded) {
                if (r.mPreserveWindow) {
                    ... // window还未显示出来，pending remove
                } else {
                    wm.removeViewImmediate(v);
                }
            }
        }
    }
    if (finishing) {
        try {
            ActivityTaskManager.getService().activityDestroyed(token);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }
    mSomeActivitiesChanged = true;
}
```
`wm.removeViewImmediate(v);` 是下一步

```java
// WindowManagerGlobal#removeView
public void removeView(View view, boolean immediate) {
    synchronized (mLock) {
        int index = findViewLocked(view, true);
        View curView = mRoots.get(index).getView();
        removeViewLocked(index, immediate);
        if (curView == view) {
            return;
        }
    }
}

private void removeViewLocked(int index, boolean immediate) {
    ViewRootImpl root = mRoots.get(index);
    View view = root.getView();

    if (root != null) {
        // 通知 InputManagerService
        root.getImeFocusController().onWindowDismissed();
    }
    boolean deferred = root.die(immediate);
    if (view != null) {
        view.assignParent(null);
        if (deferred) {
            mDyingViews.add(view);
        }
    }
}
```

下一步是 ViewRootImpl.die(boolean)

```java
boolean die(boolean immediate) {
    if (immediate && !mIsInTraversal) {
        doDie();
        return false;
    }
    ...
    // 如果不是 immediate 加入到消息循环中
    mHandler.sendEmptyMessage(MSG_DIE);
    return true;
}

void doDie() {
    checkThread();
    synchronized (this) {
        if (mRemoved) {
            return;
        }
        mRemoved = true;
        if (mAdded) {
            dispatchDetachedFromWindow();
        }

        if (mAdded && !mFirst) {
            ...
            if (mView != null) {
                // 销毁 Surface
                destroySurface();
            }
        }

        mAdded = false;
    }
    WindowManagerGlobal.getInstance().doRemoveView(this);
}


void dispatchDetachedFromWindow() {
    destroySurface();

    ....
    try {
        // binder调用移除window
        mWindowSession.remove(mWindow);
    } catch (RemoteException e) {
    }
    ...
}
```



# 如何在子线程刷新UI

因为ViewRootImpl的校验线程，只是校验ViewRootImpl实例化的线程和当前线程是否是同一个。 ViewRootImpl 是在 WindowManagerGlobal#addView 的是否创建的。 所以只要保证window的创建在子线程即可。 

```kotlin
import android.graphics.PixelFormat
import android.os.Bundle
import android.os.Handler
import android.os.HandlerThread
import android.os.Message
import android.view.WindowManager
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity
import com.example.test.R
import com.google.android.material.bottomsheet.BottomSheetDialog
import java.lang.Thread.sleep


class TestActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_test)

        val newThread = HandlerThread("work Thread")
        newThread.start()
        val newThreadHandler = object : Handler(newThread.getLooper()) {
            override fun handleMessage(msg: Message) {

                val dialog = object : BottomSheetDialog(this@TestActivity) {
                    override fun onCreate(savedInstanceState: Bundle?) {
                        super.onCreate(savedInstanceState)
                        val tvNewThreadText = TextView(this@TestActivity)
                        tvNewThreadText.setText("子线程内更新 UI ${Thread.currentThread()}")
                        setContentView(tvNewThreadText)
                    }
                }
                dialog.show()
            }
        }
        newThreadHandler.sendEmptyMessageDelayed(0, 2000)
    }
}
```


