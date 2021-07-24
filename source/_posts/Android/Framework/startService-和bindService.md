---
title: startService()和bindService()
date: 2019-07-04 20:37:58
tags: 
- Service
categories:
- Android
- Service
- Framework
---

# startService()和bindService()的区别

![](/images/startService\ and\ bindService)

### 生命周期上的差别
#### startService()
执行startService时，Service经历onCreate->onStartCommand。当执行stopService时，直接调用onDestroy方法。调用者如果没有stopService，Service会一直在后台运行，下次调用者再起来仍然可以stopService。

多次调用startService，该Service只能被创建一次，即该Service的onCreate方法只会被调用一次。**但是每次调用startService，onStartCommand方法都会被调用。**无论startService调用多少次，stopService只需要调用一次就能够终止Service

#### BindService()
bindService开启服务时，根据生命周期里onBind方法的返回值是否为空，有两种情况。
1. **onBind返回值是null**
调用bindService开启服务，生命周期执行的方法依次是：
onCreate() ==> onBind();
**调用多次bindService，onCreate和onBind也只在第一次会被执行。调用unbindService结束服务，生命周期执行onDestroy方法，并且unbindService方法只能调用一次，多次调用应用会抛出异常。** 使用时也要注意调用unbindService一定要确保服务已经开启，否则应用会抛出异常。
2. **onBind返回值不为null**
这时候调用bindService开启服务，生命周期执行的方法依次是：
onCreate() ==> onBind() ==> onServiceConnected();
可以发现我们自己写的Connection类里的onServiceConnected方法被调用了。**调用多次bindService，onCreate和onBind都只在第一次会被执行，onServiceConnected会执行多次。**
并且我们注意到onServiceConnected方法的第二个参数也是IBinder类型的，不难猜测onBind()方法返回的对象被传递到了这里。打印一下两个对象的地址可以证明猜测是正确的。
也就是说我们可以在onServiceConnected方法里拿到了Service服务的内部类Binder的对象，通过这个内部类对象，只要强转一下，我们可以调用这个内部类的非私有成员对象和方法。
调用unbindService结束服务和上面相同，unbindService只能调用一次，onDestroy也只执行一次，多次调用会抛出异常。
<br>
**总结**
第一次执行bindService时，onCreate和onBind方法会被调用，但是多次执行bindService时，onCreate和onBind方法并不会被多次调用，即并不会多次创建服务和绑定服务。

### 既使用startService又使用bindService的情况
如果一个Service又被启动又被绑定，则该Service会一直在后台运行。首先不管如何调用，onCreate始终只会调用一次。对应startService调用多少次，Service的onStart方法便会调用多少次。Service的终止，需要unbindService和stopService同时调用才行。不管startService与bindService的调用顺序，如果先调用unbindService，此时服务不会自动终止，再调用stopService之后，服务才会终止；如果先调用stopService，此时服务也不会终止，而再调用unbindService或者之前调用bindService的Context不存在了（如Activity被finish的时候）之后，服务才会自动停止。

参考链接:
[https://my.oschina.net/tingzi/blog/376545]()
[https://www.jianshu.com/p/d870f99b675c]()

# startService 启动过程源码分析

ContextImpl.startService(...) 最终会调用到 AMS 的对应方法，最终会调用到 ActivityService.startServiceLocked(...)

```java
// frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
    ComponentName startServiceLocked(IApplicationThread caller, Intent service, String resolvedType,
            int callingPid, int callingUid, boolean fgRequired, String callingPackage, final int userId)
            throws TransactionTooLargeException {

        ServiceLookupResult res =
            retrieveServiceLocked(service, resolvedType, callingPackage,
                    callingPid, callingUid, userId, true, callerFg, false, false);
        ServiceRecord r = res.record;

        ...

        // ServiceRecord.StartItem 用于在 service 启动之后调用 startCommond
        r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                service, neededGrants, callingUid));

        ComponentName cmp = startServiceInnerLocked(smap, service, r, callerFg, addToStarting);
        return cmp;
    }

    ComponentName startServiceInnerLocked(ServiceMap smap, Intent service, ServiceRecord r,
            boolean callerFg, boolean addToStarting) throws TransactionTooLargeException {

        String error = bringUpServiceLocked(r, service.getFlags(), callerFg, false, false);
        ...
        return r.name;
    }

    private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
            boolean whileRestarting, boolean permissionsReviewRequired)
            throws TransactionTooLargeException {

        if (r.app != null && r.app.thread != null) {
            // service 的进程已经启动，sendServiceArgsLocked 用于调用 startCommond() 方法
            sendServiceArgsLocked(r, execInFg, false);
            return null;
        }

        final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
        final String procName = r.processName;
        String hostingType = "service";
        ProcessRecord app;

        if (!isolated) {
            app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
            if (app != null && app.thread != null) {
                // service 不是单独的进程且，service 所在进程已经启动，则直接启动进程
                try {
                    realStartServiceLocked(r, app, execInFg);
                    return null;
                } ...
            }
        } else {
            app = r.isolatedProc;
            ...
        }

        if (app == null && !permissionsReviewRequired) {
            // servie 所在的进程还没有启动，启动相应的进程
            if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    hostingType, r.name, false, isolated, false)) == null) {
                String msg = ...
                return msg;
            }
            if (isolated) {
                r.isolatedProc = app;
            }
        }

        // 暂存进程未启动的 service , 当 service 进程启动完，Application 进行 AMS.attach() 的时候，会 mPendingServices 取出 ServiceRecord 并执行 realStartServiceLocked 
        if (!mPendingServices.contains(r)) {
            mPendingServices.add(r);
        }

        return null;
    }

    private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        if (app.thread == null) {
            // service 进程未启动，抛出异常
            throw new RemoteException();
        }
        r.app = app;
        r.restartTime = r.lastActivity = SystemClock.uptimeMillis();

        final boolean newService = app.services.add(r);

        boolean created = false;
        try {
            ...
            // 调用 Service 的 onCreate 方法
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            r.postNotification();
            created = true;
        } catch (DeadObjectException e) {
            ...
        } finally {
            ...
        }

        // 向 service 请求 binder 对象
        requestServiceBindingsLocked(r, execInFg);

        if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
            // 如果 r.pendingStarts 为空，就添加一个，pendingStarts 在调用 service.startCommond() 有用
            // 只有在 startService 的时候，r.startRequested才会为true
            r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                    null, null, 0));
        }

        // 执行 startCommond()
        sendServiceArgsLocked(r, execInFg, true);
        ...
    }
```

startService 暂时就到这，后面的就很简单了。

# bindService 启动流程源码分析

## bindService 启动流程概述

1. 应用端调用 AMS.bindService
2. AMS 查找是否存在这个 service 的 binder 对象，如果不存在，则启动 service 并请求 binder 对象
3. service 将 binder 对象发布到 AMS
4. AMS 返回 service 的 binder 对象给 应用

## 应用端

```java
// frameworks/base/core/java/android/app/ContextImpl.java
    private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags, Handler
            handler, UserHandle user) {
        IServiceConnection sd;
        if (mPackageInfo != null) {
            sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
        } else {
            throw new ...
        }
        try {
            ...
            int res = ActivityManager.getService().bindService(
                mMainThread.getApplicationThread(), getActivityToken(), service,
                service.resolveTypeIfNeeded(getContentResolver()),
                sd, flags, getOpPackageName(), user.getIdentifier());
            return res != 0;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```
IServiceConnection 是一个 IBinder 对象，但是 ServiceConnection 是一个只是一样普通对象。 ServiceDispatcher 可以理解为 IServiceConnection 和 ServiceConnection 的衔接者。 ServiceDispatcher 和 IServiceConnection 是一对一的，但是 IServiceConnection 和 ServiceConnection 不是。可以理解为 IServiceConnection 和 ServiceConnection + Context 是一对一的

我们来看 ServiceDispatcher.connected(...)

```java
// frameworks/base/core/java/android/app/LoadedApk.java
// ServiceDispatcher
        public void connected(ComponentName name, IBinder service, boolean dead) {
            if (mActivityThread != null) {
                // 从 Binder 线程池切换到主线程
                mActivityThread.post(new RunConnection(name, service, 0, dead));
            } else {
                doConnected(name, service, dead);
            }
        }
        public void doConnected(ComponentName name, IBinder service, boolean dead) {
            ServiceDispatcher.ConnectionInfo old;
            ServiceDispatcher.ConnectionInfo info;

            synchronized (this) {
                old = mActiveConnections.get(name);
                if (old != null && old.binder == service) {
                    // Huh, already have this one.  Oh well!
                    // 如果应用端已经存在 service 的 binder 对象，则不做处理。
                    // 所以，onServiceConnected 只会执行一次 
                    return;
                }

                if (service != null) {
                    // A new service is being connected... set it all up.
                    info = new ConnectionInfo();
                    info.binder = service;
                    info.deathMonitor = new DeathMonitor(name, service);
                    try {
                        // linkToDeath 监控 binder 死亡， 会调用 service.onServiceDisconnected
                        service.linkToDeath(info.deathMonitor, 0);
                        mActiveConnections.put(name, info);
                    } catch (RemoteException e) {
                        // This service was dead before we got it...  just
                        // don't do anything with it.
                        mActiveConnections.remove(name);
                        return;
                    }

                } else {
                    // The named service is being disconnected... clean up.
                    mActiveConnections.remove(name);
                }

                if (old != null) {
                    old.binder.unlinkToDeath(old.deathMonitor, 0);
                }
            }

            // If there was an old service, it is now disconnected.
            if (old != null) {
                // old 不为空，还走到了这里，说明是断开连接
                mConnection.onServiceDisconnected(name);
            }
            if (dead) {
                mConnection.onBindingDied(name);
            }
            // If there is a new viable service, it is now connected.
            if (service != null) {
                mConnection.onServiceConnected(name, service);
            } else {
                // The binding machinery worked, but the remote returned null from onBind().
                mConnection.onNullBinding(name);
            }
        }
```

## AMS

```java
    int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, final IServiceConnection connection, int flags,
            String callingPackage, final int userId) throws TransactionTooLargeException {

            ServiceLookupResult res = retrieveServiceLocked(service, ...);
            ServiceRecord s = res.record;

            if ((flags&Context.BIND_AUTO_CREATE) != 0) {
                // 启动 service ，和 bindService 一样，可以翻翻上面
                if (bringUpServiceLocked(s, service.getFlags(), callerFg, false,
                        permissionsReviewRequired) != null) {
                    return 0;
                }
            }

            if (s.app != null && b.intent.received) {
                // service 的进程已经启动，并且，已经将 binder 对象发布给 AMS
                // 直接调用应用端连接
                try {
                    c.conn.connected(s.name, b.intent.binder, false);
                } catch (Exception e) {
                    ...
                }

                // If this is the first app connected back to this binding,
                // and the service had previously asked to be told when
                // rebound, then do so.
                if (b.intent.apps.size() == 1 && b.intent.doRebind) {
                    requestServiceBindingLocked(s, b.intent, callerFg, true);
                }
            }  else if (!b.intent.requested) {
                // !b.intent.requested 表明还未向 service 请求 binder 对象
                requestServiceBindingLocked(s, b.intent, callerFg, false);
            }
    }

    private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
            boolean execInFg, boolean rebind) throws TransactionTooLargeException {
        if (r.app == null || r.app.thread == null) {
            // If service is not currently running, can't yet bind.
            return false;
        }
        if ((!i.requested || rebind) && i.apps.size() > 0) {
            // 没有请求过 binder 对象， 或者 rebind
            try {
                // 调用 service 所在对象的 onBind 并将发挥的 binder 对象发布给 AMS
                r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
                        r.app.repProcState);
                if (!rebind) {
                    i.requested = true;
                }
                i.hasBound = true;
                i.doRebind = false;
            } catch (TransactionTooLargeException e) {
                ...
            } catch (RemoteException e) {
                ...
                return false;
            }
        }
        return true;
    }
```









