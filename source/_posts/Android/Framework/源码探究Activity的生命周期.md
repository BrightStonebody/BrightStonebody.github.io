---
title: 源码探究Activity的生命周期
date: 2021-01-02 15:00:33
tags: Activity
---

## startActivity

startActivity有很多重载方法，最终都会调用`startActivityForResult`

然后的调用过程
```
-> Activit.startActivityForResult(...)
 -> Instrumentation.execStartActivity(...)
```

在 `Instrumentation.execStartActivity` 调用了 `ActivityManager.getService().startActivity(...)` ，后续的流程进入到了系统进程

```
  -> ActivityManagerService.startActivity(...)
   -> ...startActivityAsUser(...)
    -> ActivityStarter.execute()
     -> ...startActivityMayWait()
      -> ...startActivity()
       -> ...startActivityUnchecked(...)
```

`startActivityUnchecked` 是一个很重要的方法，这个方法里会根据启动标志位和Activity启动模式来决定如何启动一个Activity以及是否要调用deliverNewIntent方法通知Activity有一个Intent试图重新启动它。
因为这里只讨论住 activity 启动的主流程，所以先不看这个。

`startActivityUnchecked` 之后的调用过程如下

```
-> ActivityStarter.startActivityUnchecked(...)
 -> ActivityStackSupervisor.resumeFocusedStackTopActivityLocked(...)
  -> ActivityStack.resumeTopActivityUncheckedLocked(...)
   -> ...resumeTopActivityInnerLocked(...)
```

在 `resumeTopActivityInnerLocked` 中会先对 resume 状态的 activity 执行 pause。 

```java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    ...
    boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, next, false);
    if (mResumedActivity != null) {
        pausing |= startPausingLocked(userLeaving, false, next, false);
    }
    ...
    // 开始进行真正最终真正的activity启动
    mStackSupervisor.startSpecificActivityLocked(next, true, true);
    
    try {
        next.completeResumeLocked();
    } catch (Exception e) {
        ...
    }
    ...
    return true;
}
```

## activity 的 pause 过程

`startPausingLocked` 之后会执行 startPausingLocked

```java
final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
        ActivityRecord resuming, boolean pauseImmediately) {
    ...
    if (prev.app != null && prev.app.thread != null) {
        try {
            mService.updateUsageStats(prev, false);
            // Android 9.0在这里引入了ClientLifecycleManager和
            // ClientTransactionHandler来辅助管理Activity生命周期，
            // 注意这里是在系统进程， prev.app.thread 是应用进程的binder对象
            mService.getLifecycleManager().scheduleTransaction(prev.app.thread, prev.appToken,
                    PauseActivityItem.obtain(prev.finishing, userLeaving,
                            prev.configChangeFlags, pauseImmediately));
        } catch (Exception e) {
            ...
        }
    } else {
        mPausingActivity = null;
        mLastPausedActivity = null;
        mLastNoHistoryActivity = null;
    }
    ...
}
```

```java
void scheduleTransaction(@NonNull IApplicationThread client, @NonNull IBinder activityToken,
        @NonNull ActivityLifecycleItem stateRequest) throws RemoteException {
    final ClientTransaction clientTransaction = transactionWithState(client, activityToken,
            stateRequest);
    scheduleTransaction(clientTransaction);
}

private static ClientTransaction transactionWithState(@NonNull IApplicationThread client,
        @NonNull IBinder activityToken, @NonNull ActivityLifecycleItem stateRequest) {
    final ClientTransaction clientTransaction = ClientTransaction.obtain(client, activityToken);
    clientTransaction.setLifecycleStateRequest(stateRequest);
    return clientTransaction;
}

void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    final IApplicationThread client = transaction.getClient();
    transaction.schedule();
    if (!(client instanceof Binder)) {
        transaction.recycle();
    }
}
```

Android 9.0在这里引入了`ClientLifecycleManager`和 `ClientTransactionHandler`来辅助管理Activity生命周期，

`startPausingLocked`生成了一个 `PauseActivityItem` 然后，`ClientLifecycleManager` 会将参数打包为一个 `ClientTransaction` 并设置一个改变生命周期的request。

```java
// ClientTransaction
public void schedule() throws RemoteException {
    mClient.scheduleTransaction(this);
}
```

`schedule()` 将工作转给了 mClient ，mClient 是 transactionWithState 中传入的，是一个 IApplicationThread，这是一个APP进程的binder对象。 在 pause 流程中代表要pause的activity所在的进程。
IApplicationThread 的实现是 ActivityThread 的内部类 ApplicationThread 。这里通过binder调用从 系统进程转到了应用进程。

```java

// ApplicationThread
@Override
public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    ActivityThread.this.scheduleTransaction(transaction);
}

// ActivityThread
void scheduleTransaction(ClientTransaction transaction) {
    transaction.preExecute(this);
    sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
}

// ActivityThread
private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
    Message msg = Message.obtain();
    ...
    if (async) {
        msg.setAsynchronous(true);
    }
    mH.sendMessage(msg);
}
```

### H extends Handler

在上面可以看出，最终，向 mH 发送了一个 message

```java
    public void handleMessage(Message msg) {
        switch (msg.what) {
            ...
            case EXECUTE_TRANSACTION:
                final ClientTransaction transaction = (ClientTransaction) msg.obj;
                mTransactionExecutor.execute(transaction);
                ...
                break;
            ...
        }
    }
```

msg.what 是 `ActivityThread.H.EXECUTE_TRANSACTION` 。将activity生命周期的变换任务交给了 TransactionExecutor

```java
    public void execute(ClientTransaction transaction) {
        final IBinder token = transaction.getActivityToken();

        // 如果有callback先执行callback，后面有使用到这个特性
        executeCallbacks(transaction);
        
        executeLifecycleState(transaction);
        mPendingActions.clear();
        log("End resolving transaction");
    }

    private void executeLifecycleState(ClientTransaction transaction) {
        final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
        ...
        // Cycle to the state right before the final requested state.
        cycleToPath(r, lifecycleItem.getTargetState(), true /* excludeLastState */);

        // Execute the final transition with proper parameters.
        lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
        lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
    }
```

在 TransactionExecutor 中从 ClientTransaction 获取 ActivityLifecycleItem ， 并执行 execute 和 postExecute 。 由于我们传入的是 PauseActivityItem ， 所以真正执行的代码还是在系统进程的 PauseActivityItem 里。

```java
@Override
public void execute(ClientTransactionHandler client, IBinder token,PendingTransactionActions pendingActions) {
    // PauseActivityItem.execute 中调用的是 ClientTransactionHandler.handlePauseActivity 
    // ActivityThread 继承了 ClientTransactionHandler。 其实调用的还是 ActivityThread

    client.handlePauseActivity(token, mFinished, mUserLeaving, mConfigChanges, pendingActions,
            "PAUSE_ACTIVITY_ITEM");
}

@Override
public void postExecute(ClientTransactionHandler client, IBinder token, PendingTransactionActions pendingActions) {
    if (mDontReport) {
        return;
    }
    try {
        // TODO(lifecycler): Use interface callback instead of AMS.
        ActivityManager.getService().activityPaused(token);
    } catch (RemoteException ex) {
        throw ex.rethrowFromSystemServer();
    }
}
```

```java
// ActivityThread
@Override
public void handlePauseActivity(IBinder token, boolean finished, boolean userLeaving,
        int configChanges, PendingTransactionActions pendingActions, String reason) {
    ActivityClientRecord r = mActivities.get(token);
    if (r != null) {
        if (userLeaving) {
            performUserLeavingActivity(r);
        }

        r.activity.mConfigChangeFlags |= configChanges;
        performPauseActivity(r, finished, reason, pendingActions);

        // Make sure any pending writes are now committed.
        if (r.isPreHoneycomb()) {
            QueuedWork.waitToFinish();
        }
        mSomeActivitiesChanged = true;
    }
}

private void performPauseActivityIfNeeded(ActivityClientRecord r, String reason) {
    try {
        r.activity.mCalled = false;
        mInstrumentation.callActivityOnPause(r.activity);
    } ...
    r.setState(ON_PAUSE);
}
```

```java
// Instrumentation
public void callActivityOnPause(Activity activity) {
    activity.performPause();
}
```

## 新activity启动

```java
// ActivityStackSupervisor
    void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);

        getLaunchTimeTracker().setLaunchTime(r);

        if (app != null && app.thread != null) {
            try {
                // app.thread 不为空，表示对应的进程存在，直接启动activity
                ...
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                ...
            }
        }
        // 进程不存在，创建进程
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
    }
```

startSpecificActivityLocked 中会判断要跳转的 activity 所在的进程是否已经存在，如果存在，则调用 realStartActivityLocked ，如果没有存在，则调用 mService.startProcessLocked 创建进程。 

先看看应用进程不存在的情况。之后的调用栈如下： 
```
-> ActivityManagerService.startProcessLocked
 -> ...startProcessLocked
  -> ...startProcess
   -> Process.start
```

```
-> Process.start
 -> zygoteProcess.start
  -> ...startViaZygote
   -> ...zygoteSendArgsAndGetResult
    -> ...openZygoteSocketIfNeeded
```

```java
    private ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
        Preconditions.checkState(Thread.holdsLock(mLock), "ZygoteProcess lock not held");

        if (primaryZygoteState == null || primaryZygoteState.isClosed()) {
            try {
                primaryZygoteState = ZygoteState.connect(mSocket);
            } catch (IOException ioe) {
                throw new ZygoteStartFailedEx("Error connecting to primary zygote", ioe);
            }
            maybeSetApiBlacklistExemptions(primaryZygoteState, false);
            maybeSetHiddenApiAccessLogSampleRate(primaryZygoteState);
        }
        if (primaryZygoteState.matches(abi)) {
            return primaryZygoteState;
        }

        // The primary zygote didn't match. Try the secondary.
        if (secondaryZygoteState == null || secondaryZygoteState.isClosed()) {
            try {
                secondaryZygoteState = ZygoteState.connect(mSecondarySocket);
            } catch (IOException ioe) {
                throw new ZygoteStartFailedEx("Error connecting to secondary zygote", ioe);
            }
            maybeSetApiBlacklistExemptions(secondaryZygoteState, false);
            maybeSetHiddenApiAccessLogSampleRate(secondaryZygoteState);
        }

        if (secondaryZygoteState.matches(abi)) {
            return secondaryZygoteState;
        }

        throw new ZygoteStartFailedEx("Unsupported zygote ABI: " + abi);
    }
```

。。。这里没太看懂，但是网上的文章说 “其最终调用了Zygote并通过socket通信的方式让Zygote进程fork出一个新的进程，并根据传递的”android.app.ActivityThread”字符串，反射出该对象并执行ActivityThread的main方法对其进行初始化。“

。。。不知道他是怎么看出来的。。记个 TODO 吧

## 启动 activity

```java
// ActivityStackSupervisor
    final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
            boolean andResume, boolean checkConfig) throws RemoteException {
                ...
                // Create activity launch transaction.
                final ClientTransaction clientTransaction = ClientTransaction.obtain(app.thread,
                        r.appToken);
                // 设置 callback ，先执行 activity 的 attach 和 onCreate
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                        r.persistentState, results, newIntents, mService.isNextTransitionForward(),
                        profilerInfo));

                // Set desired final state.
                final ActivityLifecycleItem lifecycleItem;
                // 再执行 onResume
                if (andResume) {
                    lifecycleItem = ResumeActivityItem.obtain(mService.isNextTransitionForward());
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
                clientTransaction.setLifecycleStateRequest(lifecycleItem);

                // Schedule transaction. 进入app进程
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);
                ...

        return true;
    }
```

这里核心是创建了一个 ClientTransaction ，先添加了一个 LaunchActivityItem 作为 callback ， ClientTransaction 在调度是会先执行 callback 。然后设置 LifecycleStateRequest ，这里传入的 andResume == true ，所以设置的是 ResumeActivityItem 。

因为在之前 ClientTransaction 的调度流程已经分析过了，我们直接先来看 LaunchActivityItem.execute

### activity 的 onCreate

```java
// LaunchActivityItem
@Override
public void execute(ClientTransactionHandler client, IBinder token,
        PendingTransactionActions pendingActions) {
    ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
            mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
            mPendingResults, mPendingNewIntents, mIsForward,
            mProfilerInfo, client);
    client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
}

// ActivityThread
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
    ...
    final Activity a = performLaunchActivity(r, customIntent);
    ...
    return a;
}
```

```java
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        // 初始化ComponentName
        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }

        // 这个 appContext 是 activity 的 baseContext 。 不清楚作用是什么
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            // 创建 activity
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            ...
        }

        try {
            // 拿到Application
            // 在 ActivityThread#bindApplication 的使用已经实例化了applicaiton对象了。
            // 这里实际上不会创建
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (activity != null) {
                ...
                // Application、Activity和ContextImpl互相关联
                appContext.setOuterContext(activity);
                // 调用 activity 的 attach 
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                checkAndBlockForNetworkAccess();
                activity.mStartedActivity = false;
                // 设置Activity的Theme
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                // 调用 activity 的 onCreate 生命周期
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
            }
            r.setState(ON_CREATE);

            mActivities.put(r.token, r);

        } catch (SuperNotCalledException e) {
            throw e;
        } catch (Exception e) {
            ...
        }

        return activity;
    }

// Instrumentation
    public Activity newActivity(ClassLoader cl, String className,
            Intent intent)
            throws InstantiationException, IllegalAccessException,
            ClassNotFoundException {
        String pkg = intent != null && intent.getComponent() != null
                ? intent.getComponent().getPackageName() : null;
        // instantiateActivity 利用传入的 ClassLoader ，利用
        return getFactory(pkg).instantiateActivity(cl, className, intent);
    }
```

在 performLaunchActivity 中创建了 activity ，并将 context 和 application 与之绑定，执行 `activity.attach` 。 
接下来，performLaunchActivity 里通过调用 `mInstrumentation.callActivityOnCreate` ，开始进行 onCreate 的流程。

```java
// Instrumentation
    public void callActivityOnCreate(Activity activity, Bundle icicle) {
        prePerformCreate(activity);
        activity.performCreate(icicle);
        postPerformCreate(activity);
    }
```

### activity 的 onStart 

前面我们走完了 activity 的 onCreate 过程，执行代码是在 LaunchActivityItem 中，然后 onResume 显然是在 ResumeActivityItem 里， 那 onStart 呢？？

再回到 ClientTransaction 的调度流程中来， 在 TransactionExecutor 里。

```java
// TransactionExecutor
public void execute(ClientTransaction transaction) {
    final IBinder token = transaction.getActivityToken();

    executeCallbacks(transaction);

    executeLifecycleState(transaction);
    mPendingActions.clear();
}

private void executeLifecycleState(ClientTransaction transaction) {
    final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
    if (lifecycleItem == null) {
        // No lifecycle request, return early.
        return;
    }

    final IBinder token = transaction.getActivityToken();
    final ActivityClientRecord r = mTransactionHandler.getActivityClient(token);

    if (r == null) {
        // Ignore requests for non-existent client records for now.
        return;
    }

    // Cycle to the state right before the final requested state.
    cycleToPath(r, lifecycleItem.getTargetState(), true /* excludeLastState */);

    // Execute the final transition with proper parameters.
    lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
    lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
}

private void cycleToPath(ActivityClientRecord r, int finish,
        boolean excludeLastState) {
    final int start = r.getLifecycleState();
    final IntArray path = mHelper.getLifecyclePath(start, finish, excludeLastState);
    performLifecycleSequence(r, path);
}

/** Transition the client through previously initialized state sequence. */
private void performLifecycleSequence(ActivityClientRecord r, IntArray path) {
    final int size = path.size();
    for (int i = 0, state; i < size; i++) {
        state = path.get(i);
        switch (state) {
            ...
            case ON_START:
                mTransactionHandler.handleStartActivity(r, mPendingActions);
                break;
            ...
        }
    }
}
```

在 TransactionExecutor 中先执行 callback ，然后 lifecycleItem.execute 。中间有一个 cycleToPath 方法。 cycleToPath 内部逻辑很明显，执行 start 和 finish 中间状态的逻辑。 对于 ResumeActivityItem 来说就会执行 onStart 的逻辑。。。 我佛了。

activity 之后的 start 过程和 create 过程基本一致。现在再来看看 onResume

```java
// ResumeActivityItem
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        client.handleResumeActivity(token, true /* finalStateRequest */, mIsForward, "RESUME_ACTIVITY");
    }

// ActivityThread
    @Override
    public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
            String reason) {
        ...

        // 执行 activity#onResume
        final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
        ...

        final Activity a = r.activity;

        ... // 绑定 ViewRootImpl

        ...

        Looper.myQueue().addIdleHandler(new Idler());
    }
```

handleResumeActivity 中操作了很多 window 相关的东西。 到这里 activity 的启动就完成了

## activity 的 stop 和 destroy

之前分析了 onPause 的流程， 但是没有看到 onStop ， 所以 onStop 是如何执行的呢。 
onStop的执行有两种case：
1.  activity a 启动 activity b ， a.onStop 应该在 a.onPause b.onResume 之后执行。
2.  activity b 退出销毁，b.onStop 应该在 b.onPause 和 a.onResume 之后

来直接看执行puase的地方

```java
public class PauseActivityItem extends ActivityLifecycleItem {
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        client.handlePauseActivity(token, mFinished, mUserLeaving, mConfigChanges, pendingActions, "PAUSE_ACTIVITY_ITEM");
    }

    @Override
    public int getTargetState() {
        return ON_PAUSE;
    }

    @Override
    public void postExecute(ClientTransactionHandler client, IBinder token,
            PendingTransactionActions pendingActions) {
        ActivityManager.getService().activityPaused(token);
    }
}
```
poseExecute 的执行在 execute 之后； pause完后会通知给ams。

```java
// ActivityManagerService
@Override
public final void activityPaused(IBinder token) {
  synchronized(this) {
      ActivityStack stack = ActivityRecord.getStackLocked(token);
      if (stack != null) {
          stack.activityPausedLocked(token, false);
      }
  }
}

// ActivityStack
final void activityPausedLocked(IBinder token, boolean timeout) {
  final ActivityRecord r = isInStackLocked(token);
  if (r != null) {
    // 去除anr判定
      mHandler.removeMessages(PAUSE_TIMEOUT_MSG, r);
      if (mPausingActivity == r) {
          mService.mWindowManager.deferSurfaceLayout();
          try {
            // 打开新页面，进入下一步
              completePauseLocked(true /* resumeNext */, null /* resumingActivity */);
          } finally {
              mService.mWindowManager.continueSurfaceLayout();
          }
          return;
      } else {
          if (r.isState(PAUSING)) {
              r.setState(PAUSED, "activityPausedLocked");
              if (r.finishing) {
                // 退出当前页面，执行销毁流程
                  finishCurrentActivityLocked(r, FINISH_AFTER_VISIBLE, false,
                          "activityPausedLocked");
              }
          }
      }
  }
  mStackSupervisor.ensureActivitiesVisibleLocked(null, 0, !PRESERVE_WINDOWS);
}

private void completePauseLocked(boolean resumeNext, ActivityRecord resuming) {
    ActivityRecord prev = mPausingActivity;  
    if (prev != null) {
        prev.setWillCloseOrEnterPip(false);
        final boolean wasStopping = prev.isState(STOPPING); // 正常应该为false？
        prev.setState(PAUSED, "completePausedLocked");
        if (prev.finishing) {
            // 退出当前页面，执行销毁流程。 FINISH_AFTER_VISIBLE 注意这个标志位
            prev = finishCurrentActivityLocked(prev, FINISH_AFTER_VISIBLE, false,
                  "completedPausedLocked");
        } else if (prev.app != null) {
            if (mStackSupervisor.mActivitiesWaitingForVisibleActivity.remove(prev)) {
                if (prev.deferRelaunchUntilPaused) {
                    prev.relaunchActivityLocked(false /* andResume */,prev.preserveWindowOnDeferredRelaunch);
                } else if (wasStopping) {
                    prev.setState(STOPPING, "completePausedLocked");
                } else if (!prev.visible || shouldSleepOrShutDownActivities()) {
                    prev.setDeferHidingClient(false);
                    // 添加到shopping等待集合中
                    addToStopping(prev, true /* scheduleIdle */, false /* idleDelayed */);
                }
            } else {
                prev = null;
            }
            ...
            mPausingActivity = null;
        }
    }
}

final ActivityRecord finishCurrentActivityLocked(ActivityRecord r, int mode, boolean oomAdj, String reason) {
    final ActivityRecord next = mStackSupervisor.topRunningActivityLocked(true /* considerKeyguardState */);
  
    if (mode == FINISH_AFTER_VISIBLE && (r.visible || r.nowVisible) // FINISH_AFTER_VISIBLE 注意这个标记位
                  && next != null && !next.nowVisible) {
        if (!mStackSupervisor.mStoppingActivities.contains(r)) {
            addToStopping(r, false /* scheduleIdle */, false /* idleDelayed */);
        }
        r.setState(STOPPING, "finishCurrentActivityLocked");
        ...
        return r;
    }
    ...
    if (mode == FINISH_IMMEDIATELY  // FINISH_IMMEDIATELY 注意这个标记位
            || (prevState == PAUSED
                && (mode == FINISH_AFTER_PAUSE || inPinnedWindowingMode()))
            || finishingActivityInNonFocusedStack
            || prevState == STOPPING
            || prevState == STOPPED
            || prevState == ActivityState.INITIALIZING) {
        r.makeFinishingLocked();
        boolean activityRemoved = destroyActivityLocked(r, true, "finish-imm:" + reason);
        ...
        return activityRemoved ? null : r;
    }
    ...
}
```

上面的代码中，activity在pause之后，无论是finish还是非finish的流程。 都不会立刻执行 stop 和 destroy
开始执行 stop 和 destroy ，是在下一个 activity resume 之后
来看 resume 之后做了啥

```java
// ActivityThread
@Override
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,
        String reason) {
    ...

    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
    ...

    Looper.myQueue().addIdleHandler(new Idler());
}

private class Idler implements MessageQueue.IdleHandler {
    @Override
    public final boolean queueIdle() {
        ...
        if (a.activity != null && !a.activity.mFinished) {
            try {
                am.activityIdle(a.token, a.createdConfig, stopProfiling);
                    a.createdConfig = null;
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        }
        ...
        return false;
    }
}
```

在activity#resume 之后，会发送一个 idleHandler ， 在消息队列空闲时调用 ams#activityIdle 开始上一个activity的销毁流程。
因为是一个 idleHandler ，只会在空闲时执行，所以，所以有可能一个页面退出后，很长时间都不执行 stop 和 destroy。 但是系统有一个10s的兜底，10s后一定会执行 stop 和 destroy

```java
// ActivityManagerService
public final void activityIdle(IBinder token, Configuration config, boolean stopProfiling) {
    final long origId = Binder.clearCallingIdentity();
    synchronized (this) {
        ActivityStack stack = ActivityRecord.getStackLocked(token);
        if (stack != null) {
            ActivityRecord r =
                    mStackSupervisor.activityIdleInternalLocked(token, false /* fromTimeout */,
                            false /* processPausingActivities */, config);
            if (stopProfiling) {
                if ((mProfileProc == r.app) && mProfilerInfo != null) {
                    clearProfilerLocked();
                }
            }
        }
    }
    Binder.restoreCallingIdentity(origId);
}
```

```java
// ActivityStackSupervisor.java
final ActivityRecord activityIdleInternalLocked(final IBinder token, boolean fromTimeout,
        boolean processPausingActivities, Configuration config) {
    ...
    // 取出所有的之前 shopping 的 activity
    final ArrayList<ActivityRecord> stops = processStoppingActivitiesLocked(r, true /* remove */, processPausingActivities);
    ...

    for (int i = 0; i < NS; i++) {
        r = stops.get(i);
        final ActivityStack stack = r.getStack();
        if (stack != null) {
            if (r.finishing) {
                // 如果是销毁一个activity
                stack.finishCurrentActivityLocked(r, ActivityStack.FINISH_IMMEDIATELY, false, "activityIdleInternalLocked");
            } else {
                // 只是 stop
                stack.stopActivityLocked(r);
            }
        }
    }
    ...

    return r;
}

// ActivityStack
final ActivityRecord finishCurrentActivityLocked(ActivityRecord r, int mode, boolean oomAdj, String reason) {
    ...
    if (mode == FINISH_IMMEDIATELY  // 这里为true
            || (prevState == PAUSED
                && (mode == FINISH_AFTER_PAUSE || inPinnedWindowingMode()))
            || finishingActivityInNonFocusedStack
            || prevState == STOPPING
            || prevState == STOPPED
            || prevState == ActivityState.INITIALIZING) {
        r.makeFinishingLocked();
        boolean activityRemoved = destroyActivityLocked(r, true, "finish-imm:" + reason);
        ....
        return activityRemoved ? null : r;
    }
    ...
}

final boolean destroyActivityLocked(ActivityRecord r, boolean removeFromApp, String reason) {
    final boolean hadApp = r.app != null;
    ...
    if (hadApp) {
        ...
        try {
            mService.getLifecycleManager().scheduleTransaction(r.app.thread, r.appToken,
                    DestroyActivityItem.obtain(r.finishing, r.configChangeFlags));
        } catch (Exception e) {
            ...
        }
    }
    ...
    return removedFromHistory;
}

// ActivityStack
final void stopActivityLocked(ActivityRecord r) {
    ...
    mService.getLifecycleManager().scheduleTransaction(r.app.thread, r.appToken,
            StopActivityItem.obtain(r.visible, r.configChangeFlags));
    ...
}
```

先判断 `r.finishing` 如果为true，表示页面退出，执行destroy； 如果为false，则只执行stop。
经过一番调用，最终还是一样，向 ClientLifecycleManager 发送了一个 StopActivityItem 和 DestroyActivityItem。

### stop 和 destroy 系统10s兜底
在 `ActivityRecord.completeResumeLocked` 会在activity resume 之后调用，在这个方法中会调用 
`ActivityStackSuperVisor.scheduleIdleTimeoutLocked()` 
```java
void scheduleIdleTimeoutLocked(ActivityRecord next) {
    Message msg = mHandler.obtainMessage(IDLE_TIMEOUT_MSG, next);
    mHandler.sendMessageDelayed(msg, IDLE_TIMEOUT); // IDLE_TIMEOUT 是 10*1000
}

// case IDLE_TIMEOUT_MSG: {
//     activityIdleInternal((ActivityRecord) msg.obj, true /* processPausingActivities */);
//     } break;
// }
```

## anr的判定



## 参考

[（Android 9.0）Activity启动流程源码分析](https://www.jianshu.com/p/89fd44083c1c)
[Activity启动过程全解析](https://www.kancloud.cn/digest/androidframeworks/127782)
[炼狱难度！腾讯Android高级岗：为什么 Activity.finish() 之后 10s 才 onDestroy ？](https://www.jianshu.com/p/15a0df736e52)






















