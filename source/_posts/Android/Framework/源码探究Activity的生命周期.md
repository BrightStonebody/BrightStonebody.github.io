---
title: 源码探究Activity的生命周期
date: 2021-01-02 15:00:33
tags: Activity
---

## startActivity

startActivity有很多重载方法，最终都会调用`startActivityForResult`

```java
    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode, @Nullable Bundle options) {
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                // If this start is requesting a result, we can avoid making
                // the activity visible until the result is received.  Setting
                // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
                // activity hidden during this time, to avoid flickering.
                // This can only be done when a result is requested because
                // that guarantees we will get information back when the
                // activity is finished, no matter what happens to it.
                mStartedActivity = true;
            }

            cancelInputsAndStartExitTransition(options);
            // TODO Consider clearing/flushing other event sources and events for child windows.
        } else {
            ...
        }
    }
```

mParent 代表的是 ActivityGroup，是 Activity 很早的版本里才有的东西，在 api13 中已经被废弃。`mParent == null` 一定为 true

然后的调用过程
```
-> Activit.startActivity(...)
 -> Instrumentation.execStartActivity(...)
  -> ActivityManagerService.startActivity(...)
   -> ...startActivityAsUser(...)
```

## startActivityAsUser

```java
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
            boolean validateIncomingUser) {
        enforceNotIsolatedCaller("startActivity");

        userId = mActivityStartController.checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

        // TODO: Switch to user app stacks here.
        return mActivityStartController.obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setMayWait(userId)
                .execute();
    }
```

`mActivityStartController.obtainStarter` 会创建一个 ActivityStarter 

```java
    ActivityStarter setMayWait(int userId) {
        mRequest.mayWait = true;
        mRequest.userId = userId;

        return this;
    }
```

```java
    int execute() {
        try {
            if (mRequest.mayWait) {
                return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                        mRequest.callingPackage, mRequest.intent, mRequest.resolvedType,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                        mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                        mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                        mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup);
            } else {
                return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                        mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                        mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                        mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                        mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                        mRequest.outActivity, mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup);
            }
        } finally {
            onExecutionComplete();
        }
    }
```

`execute()` 中 `mRequest.mayWait` 为 true

```java
    private int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult,
            Configuration globalConfig, SafeActivityOptions options, boolean ignoreTargetSecurity,
            int userId, TaskRecord inTask, String reason,
            boolean allowPendingRemoteAnimationRegistryLookup) {
            ...
            int res = startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo,
                    voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid,
                    callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options,
                    ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason,
                    allowPendingRemoteAnimationRegistryLookup);
            ...
            return res;
        }
    }

    private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
                IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
                ActivityRecord[] outActivity) {
        ...
            result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, outActivity);
        ...
        return result;
    }
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
                EventLogTags.writeAmPauseActivity(prev.userId, System.identityHashCode(prev),
                        prev.shortComponentName, "userLeaving=" + userLeaving);
                mService.updateUsageStats(prev, false);
                // Android 9.0在这里引入了ClientLifecycleManager和
                // ClientTransactionHandler来辅助管理Activity生命周期，
                // 他会发送EXECUTE_TRANSACTION消息到ActivityThread.H里面继续处理。
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

`schedule()` 将工作转给了 mClient ，mClient 是 transactionWithState 中传入的，是一个 IApplicationThread， IApplicationThread 是 ActivityThread 的内部类。

```java

// IAppliction
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
        log("Start resolving transaction for client: " + mTransactionHandler + ", token: " + token);

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

在 TransactionExecutor 中从 ClientTransaction 获取 ActivityLifecycleItem ， 并执行 execute 和 postExecute 。 由于我们传入的是 PauseActivityItem ， 所以真正执行的代码还是在 PauseActivityItem 里。

```java
    @Override
    public void execute(ClientTransactionHandler client, IBinder token,PendingTransactionActions pendingActions) {
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
// PauseActivityItem.execute 中调用的是 ClientTransactionHandler.
// handlePauseActivity ，但是， ActivityThread 继承了 ClientTransactionHandler。 其实调用的还是 ActivityThread 。。

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

## 创建 App 进程

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
                ...
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                ...
            }
        }

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
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        // TODO: Have this take the merged configuration instead of separate global
                        // and override configs.
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                        r.persistentState, results, newIntents, mService.isNextTransitionForward(),
                        profilerInfo));

                // Set desired final state.
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                    lifecycleItem = ResumeActivityItem.obtain(mService.isNextTransitionForward());
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
                clientTransaction.setLifecycleStateRequest(lifecycleItem);

                // Schedule transaction.
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

        // 初始化ContextImpl和Activity
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
            // 初始化Application
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (activity != null) {
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }

                // 添加window
                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }
                // Application、Activity和ContextImpl互相关联
                appContext.setOuterContext(activity);
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

在 performLaunchActivity 中创建了 activity ，并将 context 和 application 与之绑定，执行 `activity.attach` 。 并添加 window ，设置主题

接下来，performLaunchActivity 里通过调用 mInstrumentation.callActivityOnCreate ，开始进行 onCreate 的流程。

```java
// Instrumentation
    public void callActivityOnCreate(Activity activity, Bundle icicle) {
        prePerformCreate(activity);
        activity.performCreate(icicle);
        postPerformCreate(activity);
    }
```

### activity 的 onResume 

前面我们走完了 activity 的 onCreate 过程，执行代码是在 LaunchActivityItem 中，然后 onResume 显然是在 ResumeActivityItem 里， 那 onOnStart 呢？？

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

        final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);
        ...

        final Activity a = r.activity;

        ...

        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            if (r.mPreserveWindow) {
                a.mWindowAdded = true;
                r.mPreserveWindow = false;
                ViewRootImpl impl = decor.getViewRootImpl();
                if (impl != null) {
                    impl.notifyChildRebuilt();
                }
            }
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                } else {
                    a.onWindowAttributesChanged(l);
                }
            }
        } else if (!willBeVisible) {
            r.hideForNow = true;
        }

        ...

        Looper.myQueue().addIdleHandler(new Idler());
    }

   public ActivityClientRecord performResumeActivity(IBinder token, boolean finalStateRequest,
            String reason) {
        ...
        try {
            r.activity.onStateNotSaved();
            r.activity.mFragments.noteStateNotSaved();
            checkAndBlockForNetworkAccess();
            if (r.pendingIntents != null) {
                deliverNewIntents(r, r.pendingIntents);
                r.pendingIntents = null;
            }
            if (r.pendingResults != null) {
                deliverResults(r, r.pendingResults, reason);
                r.pendingResults = null;
            }
            r.activity.performResume(r.startsNotResumed, reason);

            r.state = null;
            r.persistentState = null;
            r.setState(ON_RESUME);
        } catch (Exception e) {
            if (!mInstrumentation.onException(r.activity, e)) {
                throw new RuntimeException("Unable to resume activity "
                        + r.intent.getComponent().toShortString() + ": " + e.toString(), e);
            }
        }
        ...
        return r;
    }

```

handleResumeActivity 中操作了很多 window 相关的东西。 到这里 activity 的启动就完成了

## activity 的 stop

之前分析了 onPause 的流程， 但是没有看到 onStop ， 所以 onStop 是如何执行的呢。 在 activity 的生命周期中， activity a 启动 activity b ， b.onStop 应该在 a.onResume 之后执行。

玄机竟然在 handleResumeActivity 里。。

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
        final ArrayList<ActivityRecord> stops = processStoppingActivitiesLocked(r, true /* remove */, processPausingActivities);
        ...

        for (int i = 0; i < NS; i++) {
            r = stops.get(i);
            final ActivityStack stack = r.getStack();
            if (stack != null) {
                if (r.finishing) {
                    stack.finishCurrentActivityLocked(r, ActivityStack.FINISH_IMMEDIATELY, false,
                            "activityIdleInternalLocked");
                } else {
                    stack.stopActivityLocked(r);
                }
            }
        }
        ...

        return r;
    }

// ActivityStack
    final void stopActivityLocked(ActivityRecord r) {
        ...
        mService.getLifecycleManager().scheduleTransaction(r.app.thread, r.appToken,
                StopActivityItem.obtain(r.visible, r.configChangeFlags));
        ...
    }
```

经过一番调用，最终还是一样，向 ClientLifecycleManager 发送了一个 StopActivityItem 。

到此，activity 的启动流程就结束了。

## 参考

[（Android 9.0）Activity启动流程源码分析](https://www.jianshu.com/p/89fd44083c1c)
[Activity启动过程全解析](https://www.kancloud.cn/digest/androidframeworks/127782)
[炼狱难度！腾讯Android高级岗：为什么 Activity.finish() 之后 10s 才 onDestroy ？](https://www.jianshu.com/p/15a0df736e52)






















