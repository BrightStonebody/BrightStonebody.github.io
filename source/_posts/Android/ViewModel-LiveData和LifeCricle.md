---
title: 'ViewModel,LiveData和Lifecycle'
date: 2019-12-21 20:19:41
tags:
- ViewModel
---

# ViewModel、Lifecycles和LiveData
## 1. ViewModel:
ViewModel是Google官方提供的组件，是用来处理和存放和UI相关的数据的，可以把UI和数据进行分离。但就这一点来说的话，其实没有必要一定使用Google提供的ViewModel组件，我们使用自定义的ViewModel类同样可以达到这样的效果。

还有一个很重要的特性，我们都知道，在配置改变时，Activity会出现销毁重建，比如说横竖屏切换的时候。ViewModel的生命周期和Activity是不一样的，它不会被重新创建，只有当Activity退出的时候才会跟着Activity一起销毁。因此，可以将和UI相关的数据存放在ViewModel中，这样，即使Activity销毁重建，页面上显示的数据也不会丢失。
```java
public abstract class ViewModel {
    // 在ViewModel销毁的时候做回收工作，主要是为了防止ViewModel的泄漏
    protected void onCleared() {
    }
}
```
### ViewModel的创建和存储过程
主要是要搞清楚上面说的：Activity销毁重建的时候，ViewModel是如何保持不重建的。
注意，这里只是在Activity因为配置改变而销毁重建的情况下。还有一种情况是，App由前台进入后台，因为内存紧张，Activity被系统销毁。这种情况下，ViewModel是无法保存数据的。

以下全都以Activity举例，Fragment也是一样的
在Activity中获取ViewModel：
```java
viewModel = ViewModelProviders.of(this).get(XxxViewModel.class);
```
ViewModelProviders#of
```java
@NonNull
@MainThread
public static ViewModelProvider of(@NonNull FragmentActivity activity,
        @Nullable Factory factory) {
    Application application = checkApplication(activity);
    if (factory == null) {
        // 使用默认的工厂类
        // 如果想要在ViewModel的构造方法里传参，需要自定义factory类
        factory = ViewModelProvider.AndroidViewModelFactory
                                    .getInstance(application);
    }
    return new ViewModelProvider(ViewModelStores.of(activity), factory);
}
```
viewModelProvider#get
```java
// 从ViewModelStore中获取ViewModel，没有则创建
public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
    ViewModel viewModel = mViewModelStore.get(key);
    if (modelClass.isInstance(viewModel)) {
        return (T) viewModel;
    } else {
        ...
    }
    viewModel = mFactory.create(modelClass);
    mViewModelStore.put(key, viewModel);
    return (T) viewModel;
}

public class ViewModelStore {
    // ViewModel的集合，key是由ViewModel类名构成的字符串
    private final HashMap<String, ViewModel> mMap = new HashMap<>();
    ...
}
```
ViewModelStore中存储了一个Activity/Fragment的所有ViewModel。那现在问题就转变成了探究ViewModelStore的获取和存储的过程

ViewModelStores#of
```java
public static ViewModelStore of(@NonNull FragmentActivity activity) {
    if (activity instanceof ViewModelStoreOwner) {
        return ((ViewModelStoreOwner) activity).getViewModelStore();
    }
    return holderFragmentFor(activity).getViewModelStore();
}
```
这里先判断FragmentActivity是否实现了ViewModelStoreOwner接口。不同的依赖包，实现有可能是不一样的。高版本的support-v4和androidx中的Fragment和FragmentActivity都实现了这个接口。
```java
public interface ViewModelStoreOwner {
    @NonNull
    ViewModelStore getViewModelStore();
}
```
#### 实现了ViewModelStoreOwner接口

FragmentActivity
```java
public class FragmentActivity extends SupportActivity implements ViewModelStoreOwner, ...{
    private ViewModelStore mViewModelStore;
        
    public ViewModelStore getViewModelStore() {
        ...
    }
}
```

实现了ViewModelStoreOwner接口的FragmentActivity会直接持有ViewModelStore的引用

主要关注三个方法：
getViewModelStore()
onCreate()
onRetainNonConfigurationInstance()

```java
public ViewModelStore getViewModelStore() {
    if (this.getApplication() == null) {
       ...
    } else {
        if (this.mViewModelStore == null) {
            FragmentActivity.NonConfigurationInstances nc = (FragmentActivity.NonConfigurationInstances) this.getLastNonConfigurationInstance();
            if (nc != null) {
                this.mViewModelStore = nc.viewModelStore;
            }

            if (this.mViewModelStore == null) {
                this.mViewModelStore = new ViewModelStore();
            }
        }

        return this.mViewModelStore;
    }
}

protected void onCreate(@Nullable Bundle savedInstanceState) {
    ...
    FragmentActivity.NonConfigurationInstances nc = (FragmentActivity.NonConfigurationInstances)this.getLastNonConfigurationInstance();
    if (nc != null && nc.viewModelStore != null && this.mViewModelStore == null)
    {
        this.mViewModelStore = nc.viewModelStore;
    }
    ...
}
// 在页面销毁时，保存ViewModelStore
public final Object onRetainNonConfigurationInstance() {
    Object custom = onRetainCustomNonConfigurationInstance();
    ViewModelStore viewModelStore = mViewModelStore;
    if (viewModelStore == null) {
         // 如果NonConfigurationInstance保存了viewModelStore，把它取出来
        NonConfigurationInstances nc = getLastNonConfigurationInstance();
        if (nc != null) {
          viewModelStore = nc.viewModelStore;
        }
    }
    if (viewModelStore == null && custom == null) {
        return null;
    }
    NonConfigurationInstances nci = new NonConfigurationInstances();
    nci.custom = custom; 
    //把viewModelStore放到NonConfigurationInstances中并返回
    nci.viewModelStore = viewModelStore;
    //这样当页面被重建而销毁时ViewModelStore就被保存起来了。
    return nci;
}
```

ViewModelStore在onCreate里会从NonConfigurationInstances里尝试取出，然后在onRetainNonConfigurationInstanc方法中保存到NonConfigurationInstances。
onRetainNonConfigurationInstance何时被调用，数据又是怎样保存的呢？了解过Activity启动流程的都知道ActivityThread，它控制着Activity的生命周期，当ActivityThread执行performDestroyActivity这个方法时，会调用Activity#retainNonConfigurationInstances获取到保存的数据并保存到 ActivityClientRecord 中。 ActivityClientRecord 的集合 mActivities 是 ActivityThread 的成员变量，生命周期和应用进程保持一致。 

```java
ActivityClientRecord performDestroyActivity(IBinder token, boolean finishing, int configChanges, boolean getNonConfigInstance, String reason) {
   ...
   ActivityClientRecord r = mActivities.get(token);
   // 由于状态改变的destroy，这里会为true
   if (getNonConfigInstance) {
       //保存retainNonConfigurationInstances中的数据到ActivityClientRecord中
       r.lastNonConfigurationInstances = r.activity
                                          .retainNonConfigurationInstances();
   } 
   ...
   return r;
}
```

当页面重建完成,ActivityThread执行了performLaunchActivity方法时，会调用Activity的attach方法,便会把刚刚存储的数据，传递进去。

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
    activity.attach(......, r.lastNonConfigurationInstances,.....);
    ...
}
```

总结：Activity对应的ViewModel保存在ViewModelStore里，FragmentActivity持有ViewModelStore的引用，Activity因配置原因销毁-重建时，ViewModelStore被NonConfigurationInstances保存，重建时从保存处恢复。

#### 未实现ViewModelStoreOwner接口

低版本未实现ViewModelStoreOwner接口，采取的方法是在Activity中注入一个不可见的HolderFragment

```java
public static ViewModelStore of(@NonNull FragmentActivity activity) {
    if (activity instanceof ViewModelStoreOwner) {
        return ((ViewModelStoreOwner) activity).getViewModelStore();
    }
    return holderFragmentFor(activity).getViewModelStore();
}
```

HolderFragment:

```java
public class HolderFragment extends Fragment implements ViewModelStoreOwner { 
    private static final HolderFragmentManager sHolderFragmentManager = new HolderFragmentManager();
    
    private ViewModelStore mViewModelStore = new ViewModelStore();
    
    public HolderFragment() {
        setRetainInstance(true);
    }
 
    public static HolderFragment holderFragmentFor(FragmentActivity activity) {
        // 在activity中插入HolderFragment
        return sHolderFragmentManager.holderFragmentFor(activity);
    }
}
```

HolderFragment持有mViewModelStore的引用。在HolderFragment的构造方法中，调用了`setRetainInstance(true)`。
控制一个fragment实例是否在activity因为配置改变而销毁重建的时候被保留。如果设置为true, fragment在activity销毁重建的时候将会被保留。同时，fragment的生命周期将会发生改变，当Activity销毁重建时，fragment生命周期中的onCreate和onDestroy将不会被执行。

总结：HolderFragment实现了ViewModelStoreOwner接口，持有mViewModelStore的引用。由于Fragment和activity的生命周期是绑定的，Fragment内部可以感知到activity的生命周期。同时，通过setRetainInstance(true)，当activity被销毁重建的时候，HolderFragment并不会被销毁，导致mViewModelStore也不会被销毁
ViewModel的销毁：

FragmentActivity#onDestroy
```java
@Override
protected void onDestroy() {
    super.onDestroy();
    if (this.mViewModelStore != null && !this.isChangingConfigurations()) {
        //当activity由于非config改变而销毁时清空mViewModelStore
        this.mViewModelStore.clear();
    }

    this.mFragments.dispatchDestroy();
}
```

HolderFragment#onDestroy
```java
@Override
public void onDestroy() {
    super.onDestroy();
    mViewModelStore.clear();
}
```

ViewModelStore#clear()
```java
public final void clear() {
    for (ViewModel vm : mMap.values()) {
        vm.onCleared();
    }
    mMap.clear();
}
```

### ViewModel实现更进一步的数据持久化

上面的是 activity 在因为配置改变，销毁重建的情况，在这种情况下，ViewModel 可以自动实现数据保留和现场恢复。
还有一种情况是，App进入后台，然后因为内存紧张，activity 被系统销毁，然后用户再次打开，activity重建。这种情况下，ViewModel默认是不能实现现场恢复的，需要借助 SavedStateHandle。
可以简单的把 SavedStateHandle 理解为 onSaveInstanceState()的逻辑从 activity 转移到了ViewModel 中。

要想使用 SavedStateHandle 需要额外导入包：
```
implementation 'androidx.lifecycle:lifecycle-viewmodel-savedstate:2.3.0-alpha01'
```

然后在获取ViewModel时使用 SavedStateViewModelFactory
```java
viewModel = ViewModelProviders.of(this, 
                    new SavedStateViewModelFactory(getApplication(), this))
                    .get(SaveStateTestViewModel.class);
```

然后在ViewModel的构造方法里传入SavedStateHandle实例，然后就可以通过SavedStateHandle保存和读取数据了。

```java
public class SaveStateTestViewModel extends ViewModel {
    private SavedStateHandle savedStateHandle;
    private final MutableLiveData<String> notifyLiveData;

    public SaveStateTestViewModel(SavedStateHandle handle) {
        this.savedStateHandle = handle;
        this.notifyLiveData = savedStateHandle.getLiveData("key");
    }

    public void save(int data){
        savedStateHandle.set("key2", data);
    }

    public void setData(String data) {
        this.notifyLiveData.setValue(data);
    }

    public MutableLiveData<String> getNotifyLiveData(){
        return notifyLiveData;
    }
}
```

和 onSaveInstanceState() 的保存和恢复方式一样，不能保存复杂的数据，只能保存可以存入Bundle的那些类型。 也可以SavedStateHandle中使用LiveData的方式进行保存，这样的话稍微有那么一点的好处是你不需要特意的save和get数据，直接把在初始化的时候替换成SavedStateHandle#getLiveData就可以了

## 2. Lifecycles

Lifecycles组件是为了方便感知Activity和Fragment的生命周期而出现的，它可以让任何一个类都能轻松感知到Activity的生命周期，同时又不需要在Activity中编写大量的逻辑处理。
Lifecycle包含在support library 26.1.0及之后的依赖包中，如果我们的项目基于这些依赖包，那么不需要额外的引用。support library在26.1.0之前，需要我们引入另外的包。

lifecycle 组件包括 Lifecycle, LifecycleOwner, LifecycleObserver 三个主要类：
  - Lifecycle 类持有 Fragment 和 Activity (LifecycleOwner) 的生命周期状态信息，并且允许其他对象(LifecycleObserver)观察这些信息。
  - LifecycleOwner 仅有一个 getLifecycle() 方法，用于返回当前对象的 Lifecycle。
  - LifecycleObserver，代表观察者

```java
LifecyclerOwner和LifecycleObserver
public interface LifecycleOwner {
    Lifecycle getLifecycle();
}

public interface LifecycleObserver {
}
```

LifecycleOwner是Lifecycle组件包中的一个接口，所有需要管理生命周期的类型都必须实现这个接口，是被观察者。
LifecycleOwner#getLifecycle()返回Lifecycle实例. 这是一种解耦实现, LifecycleOwner不包含任何有关生命周期管理的逻辑, 实际的逻辑都在Lifecycle实例中, 我们可以通过传递Lifecycle实例而非LifecycleOwner来防止内存泄漏。
并不是所有的google官方的Activity和Fragment类都实现了LifecycleOwner接口, 实现了这个接口有FragmentActivity，AppCompatActivity和v4包下的Fragment。
LifecycleObserver接口代表生命周期的观察者，它的实现为空。实现LifecycleObserver的类要想感知到生命周期的变化需要借助额外的注解。或者，也可以使用一些继承了LifecycleObserver的子接口。

```java
public class MyObserver implements LifecycleObserver {     
    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)    
    public voidactivityResume() {
    } 
}
```

### Lifecycle中的state和event

```java
public enum State {
    DESTROYED,
    INITIALIZED,
    CREATED,
    STARTED,
    RESUMED;
    public boolean isAtLeast(@NonNull State state) {
        return compareTo(state) >= 0;
    }
}

public enum Event {
    ON_CREATE,
    ON_START,
    ON_RESUME,
    ON_PAUSE,
    ON_STOP,
    ON_DESTROY,
    ON_ANY // ON_ANY代表任意时间
}
```

### Lifecycle的获取生命周期的变化

```java
public class SupportActivity extends Activity implements LifecycleOwner, Component {
    private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
    
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ReportFragment.injectIfNeededIn(this);
    }

    protected void onSaveInstanceState(Bundle outState) {
        this.mLifecycleRegistry.markState(State.CREATED);
        super.onSaveInstanceState(outState);
    }

    public Lifecycle getLifecycle() {
        return this.mLifecycleRegistry;
    }
}
```

LifecycleRegistry继承了Lifecycle。SupportActivity是FragmentActivity的父类。在SupportActivity中找不到state和event分发相关的代码，这些相关内容是在ReportFragment中实现的。

```java
public class ReportFragment extends Fragment {
    public static void injectIfNeededIn(Activity activity) {
        android.app.FragmentManager manager = activity.getFragmentManager();
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();

            manager.executePendingTransactions();
        }
    }
    
    @Override
    public void onStart() {
        super.onStart();
        dispatchStart(mProcessListener);
        dispatch(Lifecycle.Event.ON_START);
    }
    
    private void dispatch(Lifecycle.Event event) {
        Activity activity = getActivity();
        ...
        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }
    ...
}
```
### LifecycleRegistry#addObserver

LifecycleRegistry继承了Lifecycle，是Activity和Fragment生命周期的真正管理类。 这个类里面的很多逻辑有点绕，我不太能理清楚。主要看一下addObserver这个方法：

```java
public class LifecycleRegistry extends Lifecycle {

    private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap = new FastSafeIterableMap<>();
    private final WeakReference<LifecycleOwner> mLifecycleOwner;
    // 当前的state
    private State mState;

    @Override
    public void addObserver(@NonNull LifecycleObserver observer) {
        // observer的state初始化为DESTROYED或者INITIALIZED
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);
    
        if (previous != null) {
            return;
        }
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            // it is null we should be destroyed. Fallback quickly
            return;
        }
    
        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
        // 得到一个目标state
        State targetState = calculateTargetState(observer);
        mAddingObserverCounter++;
        // 当observer被add进来的时候，会进入循环，分发事件，直到observer的state==targetState
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
            pushParentState(statefulObserver.mState);
            // 这里的upEvent(state)会导致statefulObserver.mState在分发的过程中不断更新
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            popParentState();
            // mState / subling may have been changed recalculate
            targetState = calculateTargetState(observer);
        }
    
        if (!isReentrance) {
            // we do sync only on the top level.
            sync();
        }
        mAddingObserverCounter--;
    }

    private State calculateTargetState(LifecycleObserver observer) {
        // 这里的previous指前一个加入进来的Observer
        Entry<LifecycleObserver, ObserverWithState> previous = mObserverMap.ceil(observer);
    
        State siblingState = previous != null ? previous.getValue().mState : null;
        // 这个mParentStates不知道是啥。。。
        State parentState = !mParentStates.isEmpty() ? mParentStates.get(mParentStates.size() - 1)
                : null;
        // 这里的min(x, y)是 返回在枚举中order最小的state
        return min(min(mState, siblingState), parentState);
    }
    ...
}
```

在addObserver中，ObserverWithState将state和observer包装一下，ObserverWithState内初始的state是DESTROYED或者INITIALIZED。之后会计算一个targetState，在while循环中会不断的分发和更新event。

具体效果看下面的demo：

```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    ...
    handler.postDelayed(new Runnable() {
        @Override
        public void run() {
            TestActivity.this.getLifecycle().addObserver(new GenericLifecycleObserver() {
                @Override
                public void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event) {
                    Log.i("chenlei", "onStateChanged: " + event);
                }
            });
        }
    }, 5000);
}
```

这里的GenericLifecycleObserver继承了LifecycleObserver的接口。demo中在create方法里延时5s添加一个LifecycleObserver并打印event。最后打印输出如下：

可以看到从ON_CREATE到ON_RESUME在同一时刻都打印出来了

### ProcessLifecycleOwner

ProcessLifecycleOwner实现了LifecycleOwner接口，可以监听当前应用前后台切换，并对此进行响应。
这个类提供整个应用进程的生命周期。
你可以将这个 LifecycleOwner 作为全体 Activity 的 LifecycleOwner，Lifecycle.Event.ON_CREATE 事件只会分发一次，而 Lifecycle.Event.ON_DESTROY 永远不被分发。其他事件的分发遵守以下规则：
ProcessLifecycleOwner 会在第一个 Activity 经历 Lifecycle.Event.ON_START 和 Lifecycle.Event.ON_RESUME 事件时分发这些事件。 ProcessLifecycleOwner 会在最后一个 Activity 经历 Lifecycle.Event.ON_PAUSE 和 Lifecycle.Event.ON_STOP 事件 延迟一段时间 分发这些事件。这个延时足够长以确保 ProcessLifecycleOwner 不会在 Activity 由于配置变更而销毁重建期间发送任何事件。
这个类非常适用于对 app 前后台状态切换时进行响应、且对生命周期不要求毫秒级准确性的场景。
使用的时候，在Application中添加LifecycleObserver就可以了(实际上，在应用进程中的任意位置都可以）

```java
class ApplicationObserver(val analytics: Analytics) : LifecycleObserver {
    
    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    fun onBackground() {    }   
    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    fun onForeground() {    }
}

public class MyApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();

        ProcessLifecycleOwner.get().getLifecycle().addObserver(new ApplicationObserver());
    }
}
```

原理：

ProcessLifecycleOwnerInitializer
```java
public class ProcessLifecycleOwnerInitializer extends ContentProvider {
    @Override
    public boolean onCreate() {
        LifecycleDispatcher.init(getContext());
        ProcessLifecycleOwner.init(getContext());
        return true;
    }
    ...
}
```

这个类是ProcessLifecycleOwner初始化的地方，它继承了一个ContentProvider ，这个ContentProvider声明在了Lifecycle包的AndroidManifest.xml里。

这个AndroidManifest.xml最终会合并入我们app module 的AndroidManifest.xml文件中。
这里利用了 ContentProvider 的隐式加载。它的 onCreate() 方法执行时机是在Application 的 onCreate()方法之前。这样它就能通过 Application 来监听 Activity 的创建。

LifecycleDispatcher#init
```java
class LifecycleDispatcher {

    private static AtomicBoolean sInitialized = new AtomicBoolean(false);

    static void init(Context context) {
        if (sInitialized.getAndSet(true)) {
            return;
        }
        ((Application) context.getApplicationContext())
                .registerActivityLifecycleCallbacks(new DispatcherActivityCallback());
    }

    static class DispatcherActivityCallback extends EmptyActivityLifecycleCallbacks {

        @Override
        public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
            ReportFragment.injectIfNeededIn(activity);
        }
        ...
    }
}
```

在Application中注册一个ActivityLifecycleCallbacks，在ActivityLifecycleCallbacks中有Activity所有生命周期的回调，这样就能监听到每一个Activity的创建和销毁。在onActivityCreated回调中往Activity注入一个了ReportFragment，我没看出来这里的ReportFragment的作用是什么。。

ProcessLifecycleOwner

```java
public class ProcessLifecycleOwner implements LifecycleOwner {
    private final LifecycleRegistry mRegistry = new LifecycleRegistry(this);
    private static final ProcessLifecycleOwner sInstance = new ProcessLifecycleOwner();
    static void init(Context context) {
        sInstance.attach(context);
    }

    private ActivityInitializationListener mInitializationListener =
        new ActivityInitializationListener() {
            @Override
            public void onCreate() {
            }

            @Override
            public void onStart() {
                activityStarted();
            }

            @Override
            public void onResume() {
                activityResumed();
            }
        };

    void attach(Context context) {
        mHandler = new Handler();
        mRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);
        Application app = (Application) context.getApplicationContext();
        app.registerActivityLifecycleCallbacks(new EmptyActivityLifecycleCallbacks() {
            @Override
            public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
            //在ReportFragment的生命周期中会有dispatchXxxx(mProcessListener)
                ReportFragment.get(activity)
                                .setProcessListener(mInitializationListener);
            }
    
            @Override
            public void onActivityPaused(Activity activity) {
                activityPaused();
            }
    
            @Override
            public void onActivityStopped(Activity activity) {
                activityStopped();
            }
        });
    }

    void activityResumed() {
        mResumedCounter++;
        if (mResumedCounter == 1) {
            if (mPauseSent) {
                mRegistry.handleLifecycleEvent(Lifecycle.Event.ON_RESUME);
                mPauseSent = false;
            } else {
                mHandler.removeCallbacks(mDelayedPauseRunnable);
            }
        }
    }

    void activityPaused() {
        mResumedCounter--;
        if (mResumedCounter == 0) {
            mHandler.postDelayed(mDelayedPauseRunnable, TIMEOUT_MS);
        }
    }
    // 在mDelayedPauseRunnable中会调用这个方法
    private void dispatchPauseIfNeeded() {
        if (mResumedCounter == 0) {
            mPauseSent = true;
            mRegistry.handleLifecycleEvent(Lifecycle.Event.ON_PAUSE);
        }
    }
    ...
}
```

## 2. LiveData
ViewModel和Lifecycles分别代表数据和UI两端，这两个组件是相对比较独立的，并没有太多直接的关系。为了防止内存泄漏，ViewModel不能直接持有activity和fragment的引用。为了让更好的结合ViewModel和Lifecycles，就需要LiveData。
LiveData是一种观察者模式, 在 Value发生变化时通知之前注册的所有观察者。同时，借助于Lifecycles，LiveData具有生命周期的感知能力。如果观察者（由 Observer 类表示）的生命周期处于 STARTED 或 RESUMED 状态，则 LiveData 会认为该观察者处于活跃状态。使用LiveData有以下的优点：
- LiveData 只会将消息通知给活跃的观察者，非活跃观察者将不会收到消息通知。
- 当非活跃观察者状态变为活跃的时候，LiveData会将自动将最新的消息发送给它。
- 当相应的 Lifecycle 对象的状态变为 DESTROYED 时，LiveData会自动的移除此观察者，这样就不用担心Fragment和Activity的内存泄漏。
- LiveData的数据发送和Observer的添加是没有顺序限制的。比如说LiveData在第1s发送消息，然后在第5s添加Observer，Observer依然能够立即接收到第1s发送出的数据，只要数据是当前最新的。这主要和LifecycleRegistry#addObserver的实现有关

### 2.1 LiveData的使用

#### 2.1.1 基本使用

```java
 public class NameViewModel extends ViewModel {
    private MutableLiveData<String> notifyCurrentName = new MutableLiveData<>();

    private void post(){
        notifyCurrentName.postValue("value");    
    }
    
    public MutableLiveData<String> getNotifyCurrentName(){
        return notifyCurrentName;
    }
}

// 在activity or fragment中
// public void observe(LifecycleOwner owner, Observer<T> observer) 
getViewModel().getNotifyCurrentName().observer(this, new Observer<String>() {
    @Override
    public void onChanged(@Nullable Stirng value) {
        ...
    }
}

// 如果不想让LiveData感知到生命周期，可以用observeForever(Observer<T> observer)
getViewModel().getNotifyCurrentName().observeForever(new Observer<String>() {
    @Override
    public void onChanged(@Nullable Stirng value) {
        ...
    }
}
```

#### 2.1.2 Transformations#map
Transformations是用来做LiveData转换的类。
Transformations#map可以在消息分派给观察者之前对消息进行更改。

```java
LiveData<User> userLiveData = ...;
LiveData<String> userName = Transformations.map(userLiveData, user -> {
    user.name + " " + user.lastName
});
```

#### 2.1.3 Transformations#switchMap
switchMap可以实现LiveData的转换，下面是一个demo，通过switchMap切换LiveData的数据源

```java
public class MapTestViewModel extends ViewModel {
    private MutableLiveData<String> dataSourceA = new MutableLiveData<>();
    private MutableLiveData<String> dataSourceB = new MutableLiveData<>();

    private MutableLiveData<Boolean> changeDataSource = new MutableLiveData<>();

    private LiveData<String> notifyData = Transformations.switchMap(changeDataSource, new Function<Boolean, LiveData<String>>() {
        @Override
        public LiveData<String> apply(Boolean input) {
            return input ? dataSourceA : dataSourceB;
        }
    });

    public void start(Activity activity) {
        changeDataSource.setValue(true);

        Handler handler = new Handler(activity.getMainLooper());
        handler.postDelayed(() -> {
            dataSourceA.postValue("data from source A");
            dataSourceB.postValue("data from source B");
        }, 2000);
        
        handler.postDelayed(() -> {
            changeDataSource(false);
        }, 10000);
    }

    public void changeDataSource(boolean userA) {
        changeDataSource.setValue(userA);
    }

    public LiveData<String> getNotifyData() {
        return notifyData;
    }
}

// activity
viewModel.start(this);
viewModel.getNotifyData().observe(MapTestActivity.this, new Observer<String>() {
    @Override
    public void onChanged(String s) {
        Log.i("chenlei", "onChanged: " + s);
    }
});
```
            
上面的demo中，延时2s同时向dataSourceA和dataSourceB发送数据，并延时10s将数据源从A指向B。在observe中打印结果，最终dataSourceA和dataSourceB在间隔一段时间之后都能打印出来。
每次调用changeDataSource.setValue(Boolean)，都会调用Function#apply返回一个LiveData，并将notifyData的 Observer 添加到新的LiveData上

### 2.2 LiveData的原理

#### 2.2.1 LiveData

```java
public abstract class LiveData<T> {
    // 存放Observer的集合
    private SafeIterableMap<Observer<T>, ObserverWrapper> mObservers =                                                                 new SafeIterableMap<>();

    static final int START_VERSION = -1;
    private int mActiveCount = 0;
    private int mVersion = START_VERSION;

    ...
    // 当LiveData的活跃Observer数量由0变成1的时候调用
    protected void onActive() {}
    // 当LiveData的活跃Observer数量由1变成0的时候调用
    protected void onInactive() {}

    ...

    private abstract class ObserverWrapper {
        final Observer<T> mObserver;
        boolean mActive;
        int mLastVersion = START_VERSION;
    
        ObserverWrapper(Observer<T> observer) {
            mObserver = observer;
        }
    
        abstract boolean shouldBeActive();
    
        boolean isAttachedTo(LifecycleOwner owner) {
            return false;
        }
    
        void detachObserver() {
        }
    
        void activeStateChanged(boolean newActive) {
            // 活跃状态没有改变则return， 防止重复发送消息
            if (newActive == mActive) {
                return;
            }
            mActive = newActive;
            boolean wasInactive = LiveData.this.mActiveCount == 0;
            LiveData.this.mActiveCount += mActive ? 1 : -1;
            if (wasInactive && mActive) {
                onActive();
            }
            if (LiveData.this.mActiveCount == 0 && !mActive) {
                onInactive();
            }
            if (mActive) {
                // 开始分发value
                dispatchingValue(this);
            }
        }
    }
    ...
}
```

#### 2.2.2 LiveData#observer

```java
public abstract class LiveData<T> {    
    ...
    @MainThread
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
        if (owner.getLifecycle().getCurrentState() == DESTROYED) {
            return;
        }
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        ...
        if (existing != null) {
            return;
        }
        // 将wrapper将入到lifecycleOwner的生命周期的监听中
        owner.getLifecycle().addObserver(wrapper);
    }
    ...
}
```

LifecycleBoundObserver继承自ObserverWrapper，并且实现了GenericLifecycleObserver接口。GenericLifecycleObserver接口继承自LifecycleObserver接口

```java
public interface GenericLifecycleObserver extends LifecycleObserver {
    void onStateChanged(LifecycleOwner source, Lifecycle.Event event);
}
```

```java
class LifecycleBoundObserver extends ObserverWrapper implements GenericLifecycleObserver {
    @NonNull final LifecycleOwner mOwner;

    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<T> observer)
    {
        super(observer);
        mOwner = owner;
    }

    @Override
    boolean shouldBeActive() {
        // 通过Lifecycle判断当前Observer是否活跃
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }

    // GenericLifecycleObserver#onStatechanged
    @Override
    public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
        if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
            // 当LifecycleOwner已经被销毁，remove这个Observer。
            // removeObserver会调用detachObserver()和activeStateChanged(false)
            // 这也是为什么LiveData可以自动防止UI泄漏的原因
            removeObserver(mObserver);
            return;
        }
        activeStateChanged(shouldBeActive());
    }

    @Override
    boolean isAttachedTo(LifecycleOwner owner) {
        return mOwner == owner;
    }

    @Override
    void detachObserver() {
        // 从LifecycleOwner中remove观察者
        mOwner.getLifecycle().removeObserver(this);
    }
}
```

LiveData#observeForever

```java
@MainThread
public void observeForever(@NonNull Observer<T> observer) {
    AlwaysActiveObserver wrapper = new AlwaysActiveObserver(observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    ...
    if (existing != null) {
        return;
    }
    // 因为observeForever没有LifecycleOwner，为了保持和普通observe方法的特性一致，在添加observe之后立刻检查是否有数据可以发送，
    wrapper.activeStateChanged(true);
}

private class AlwaysActiveObserver extends ObserverWrapper {
    AlwaysActiveObserver(Observer<T> observer) {
        super(observer);
    }

    @Override
    boolean shouldBeActive() {
        return true;
    }
}
```

#### 2.2.3 LiveData#postValue()
LiveData发送消息使用setValue或者postValue方法。其中setValue只能在主线程中使用，postValue发送消息时会自动切换到主线程。

```java
public abstract class LiveData<T> {
    private final Runnable mPostValueRunnable = new Runnable() {
        @Override
        public void run() {
            ...
            setValue((T) newValue);
        }
    };

    ... 
    protected void postValue(T value) {
        ...
        ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
    }
    ...
}
LiveData#setValue
@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    dispatchingValue(null);
}

private void dispatchingValue(@Nullable ObserverWrapper initiator) {
    if (mDispatchingValue) {
        mDispatchInvalidated = true;
        return;
    }
    mDispatchingValue = true;
    do {
        mDispatchInvalidated = false;
        if (initiator != null) {
            // 向特定的Observer发送数据
            considerNotify(initiator);
            initiator = null;
        } else {
            // 向所有Observer发送消息
            for (Iterator<Map.Entry<Observer<T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                considerNotify(iterator.next().getValue());
                if (mDispatchInvalidated) {
                    break;
                }
            }
        }
    } while (mDispatchInvalidated);
    mDispatchingValue = false;
}

private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        return;
    }
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    if (observer.mLastVersion >= mVersion) {
        // 将要发送的是旧版本，扔掉
        return;
    }
    observer.mLastVersion = mVersion;
    observer.mObserver.onChanged((T) mData);
}
```
