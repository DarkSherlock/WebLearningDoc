android系统开机，Linux <init>进程启动后从init进程fork出来zygote进程，zygote开启的时候，会调用ZygoteInit.main()进行初始化，然后开始fork system_server进程。

    public static void main(String argv[]) {
    
     ...ignore some code...
    
    //在加载首个zygote的时候，会传入初始化参数，使得startSystemServer = true
     boolean startSystemServer = false;
    for (int i = 1; i < argv.length; i++) {
    	if ("start-system-server".equals(argv[i])) {
    		startSystemServer = true;
    	} else if (argv[i].startsWith(ABI_LIST_ARG)) {
    		abiList = argv[i].substring(ABI_LIST_ARG.length());
    	} else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
    		socketName = argv[i].substring(SOCKET_NAME_ARG.length());
    	} else {
    		throw new RuntimeException("Unknown command line argument: " + argv[i]);
    	}
    }
  
    ...ignore some code...
    
     //开始fork我们的SystemServer进程
     if (startSystemServer) {
    	startSystemServer(abiList, socketName);
     }
    
     ...ignore some code...
    }

看看startSystemServer（）具体怎么执行的

    private static boolean startSystemServer(String abiList, String socketName) throws MethodAndArgsCaller, RuntimeException {
    
     ...ignore some code...
    
    //留着这段注释，就是为了说明上面ZygoteInit.main(String argv[])里面的argv就是通过这种方式传递进来的
    /* Hardcoded command line to start the system server */
	    String args[] = {
	    "--setuid=1000",
	    "--setgid=1000",
	    "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1032,3001,3002,3003,3006,3007",
	    "--capabilities=" + capabilities + "," + capabilities,
	    "--runtime-init",
	    "--nice-name=system_server",
	    "com.android.server.SystemServer",
	    };
    
	    int pid;
	    try {
		    parsedArgs = new ZygoteConnection.Arguments(args);
		    ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
		    ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);
	    
		    //确实是fork出来的
		    /* Request to fork the system server process */
		    pid = Zygote.forkSystemServer(
		    parsedArgs.uid, parsedArgs.gid,
		    parsedArgs.gids,
		    parsedArgs.debugFlags,
		    null,
		    parsedArgs.permittedCapabilities,
		    parsedArgs.effectiveCapabilities);
	    } catch (IllegalArgumentException ex) {
	    	throw new RuntimeException(ex);
	    }
    
	    /* For child process */
	    if (pid == 0) {
		    if (hasSecondZygote(abiList)) {
		    	waitForSecondaryZygote(socketName);
		    }
	    
	    	handleSystemServerProcess(parsedArgs);
	    }
	    
	    return true;
    }
system_server进程开启后，会初始化系统的其他服务电源、AMS等。

    public final class SystemServer {
    
	    //zygote的主入口
	    public static void main(String[] args) {
	    	new SystemServer().run();
	    }
    
	    public SystemServer() {
	    	// Check for factory test mode.
	    	mFactoryTestMode = FactoryTest.getMode();
	    }
    
	    private void run() {
	    
	    	...ignore some code...
	    
	    	//加载本地系统服务库，并进行初始化 
	    	System.loadLibrary("android_servers");
	    	nativeInit();
	    
	    	// 创建系统上下文
	    	createSystemContext();
	    
		    //初始化SystemServiceManager对象，下面的系统服务开启都需要调用SystemServiceManager.startService(Class<T>)，这个方法通过反射来启动对应的服务
		    mSystemServiceManager = new SystemServiceManager(mSystemContext);
	    
	    	//开启服务
	    	try {
	    		startBootstrapServices();
			    startCoreServices();
			    startOtherServices();
	    	} catch (Throwable ex) {
			    Slog.e("System", "******************************************");
			    Slog.e("System", "************ Failure starting system services", ex);
	    		throw ex;
	    	}
	    
	    	...ignore some code...
	    
	    }
	    
	    //初始化系统上下文对象mSystemContext，并设置默认的主题,mSystemContext实际上是一个ContextImpl对象。调用ActivityThread.systemMain()的时候，会调用ActivityThread.attach(true)，而在attach()里面，则创建了Application对象，并调用了Application.onCreate()。
	    private void createSystemContext() {
	    	ActivityThread activityThread = ActivityThread.systemMain();
	    	mSystemContext = activityThread.getSystemContext();
	    	mSystemContext.setTheme(android.R.style.Theme_DeviceDefault_Light_DarkActionBar);
	    }
	    
	    //在这里开启了几个核心的服务，因为这些服务之间相互依赖，所以都放在了这个方法里面。
	    private void startBootstrapServices() {
	    
	    	...ignore some code...
	    
		    //初始化ActivityManagerService
		    mActivityManagerService = mSystemServiceManager.startService(
		    ActivityManagerService.Lifecycle.class).getService();
		    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
	    
		    //初始化PowerManagerService，因为其他服务需要依赖这个Service，因此需要尽快的初始化
		    mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
	    
		    // 现在电源管理已经开启，ActivityManagerService负责电源管理功能
		    mActivityManagerService.initPowerManagement();
	    
		    // 初始化DisplayManagerService
		    mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);
	    
		    //初始化PackageManagerService
		    mPackageManagerService = PackageManagerService.main(mSystemContext, mInstaller,
	        mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
	    
	    	...ignore some code...
	    
	    }
    }

经过上面这些步骤，我们的ActivityManagerService对象已经创建好了，并且完成了成员变量初始化。而且在这之前，调用createSystemContext()创建系统上下文的时候，也已经完成了mSystemContext和ActivityThread的创建。注意，这是系统进程开启时的流程，在这之后，会开启系统的Launcher程序，完成系统界面的加载与显示。

你是否会好奇，我为什么说AMS是服务端对象？下面我给你介绍下Android系统里面的服务器和客户端的概念。

其实服务器客户端的概念不仅仅存在于Web开发中，在Android的框架设计中，使用的也是这一种模式。服务器端指的就是所有App共用的系统服务，比如我们这里提到的ActivityManagerService，和前面提到的PackageManagerService、WindowManagerService等等，这些基础的系统服务是被所有的App公用的，当某个App想实现某个操作的时候，要告诉这些系统服务，比如你想打开一个App，那么我们知道了包名和MainActivity类名之后就可以打开。

但是，我们的App通过调用startActivity()并不能直接打开另外一个App，这个方法会通过一系列的调用，最后还是告诉AMS说：“我要打开这个App，我知道他的住址和名字，你帮我打开吧！”所以是AMS来通知zygote进程来fork一个新进程，来开启我们的目标App的。这就像是浏览器想要打开一个超链接一样，浏览器把网页地址发送给服务器，然后还是服务器把需要的资源文件发送给客户端的。

知道了Android Framework的客户端服务器架构之后，我们还需要了解一件事情，那就是我们的App和AMS(SystemServer进程)还有zygote进程分属于三个独立的进程，他们之间如何通信呢？

App与AMS通过Binder进行IPC通信，AMS(SystemServer进程)与zygote通过Socket进行IPC通信。

那么AMS有什么用呢？在前面我们知道了，如果想打开一个App的话，需要AMS去通知zygote进程，除此之外，其实所有的Activity的开启、暂停、关闭都需要AMS来控制，所以我们说，AMS负责系统中所有Activity的生命周期。

当在桌面上点击App启动新的App时，其实是触发了Launcher的startActivity，Launcher本身也是一个Activity，所以其实和我们在Activity中直接startActivity()基本一样，都是调用了Activity.startActivityForResult()。

每个Activity都持有Instrumentation对象的一个引用，但是整个进程只会存在一个Instrumentation对象。当startActivityForResult()调用之后，实际上还是调用了mInstrumentation.execStartActivity()

    public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
    	if (mParent == null) {
    		Instrumentation.ActivityResult ar =
    			mInstrumentation.execStartActivity(
		    		this, mMainThread.getApplicationThread(), mToken, this,
		    		intent, requestCode, options);
	    	if (ar != null) {
	    		mMainThread.sendActivityResult(
	    		mToken, mEmbeddedID, requestCode, ar.getResultCode(),
	    		ar.getResultData());
	    	}
    		...ignore some code...
	    } else {
		    if (options != null) {
		     	//当现在的Activity有父Activity的时候会调用，但是在startActivityFromChild()内部实际还是调用的mInstrumentation.execStartActivity()
		    	mParent.startActivityFromChild(this, intent, requestCode, options);
		    } else {
		    	mParent.startActivityFromChild(this, intent, requestCode);
		    }
	    }
     	...ignore some code...
    }


下面是mInstrumentation.execStartActivity()的实现：

	public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
            ...ignore some code...
      try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess();
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
        }
        return null;
    }
ActivityManagerNative的实现

	public abstract class ActivityManagerNative extends Binder implements IActivityManager{

		//从类声明上，我们可以看到ActivityManagerNative是Binder的一个子类，而且实现了IActivityManager接口
		static public IActivityManager getDefault() {
	        return gDefault.get();
	    }
	
	    //通过单例模式获取一个IActivityManager对象，这个对象通过asInterface(b)获得
	    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
	        protected IActivityManager create() {
	            IBinder b = ServiceManager.getService("activity");
	            if (false) {
	                Log.v("ActivityManager", "default service binder = " + b);
	            }
	            IActivityManager am = asInterface(b);
	            if (false) {
	                Log.v("ActivityManager", "default service = " + am);
	            }
	            return am;
	        }
	    };
	    }
	
	
		//最终返回的还是一个ActivityManagerProxy对象
		static public IActivityManager asInterface(IBinder obj) {
	        if (obj == null) {
	            return null;
	        }
	        IActivityManager in =
	            (IActivityManager)obj.queryLocalInterface(descriptor);
	        if (in != null) {
	            return in;
	        }
	
	     	//这里面的Binder类型的obj参数会作为ActivityManagerProxy的成员变量保存为mRemote成员变量，负责进行IPC通信
	        return new ActivityManagerProxy(obj);
	    }
	}

 //如果App进程没有启动的话，启动进程
启动进程的操作会先调用AMS.startProcessLocked方法，内部调用 Process.start(android.app.ActivityThread);而后通过socket通信告知Zygote进程fork子进程，即app进程。进程创建后将ActivityThread加载进去，执行ActivityThread.main()方法。

main方法会实例化ActivityThread，同时创建ApplicationThread，Looper，Hander对象，调用attach方法进行Binder通信，looper启动循环。attach方法内部获取ActivityManagerService代理对象，作为客户端调用attachApplication(mAppThread)方法，将thread信息告知AMS。

1.AMS中真正调用的是attachApplicationLocked方法，调用thread.bindApplication。thread是之前Process.start创建出来的App进程中ActivityThread中的成员变量ApplicationThread，通过aidl（IBinder)和AMS通信。

回到ActivityThread, bindApplication（）方法中会通过Handler发送BIND_APPLICATION消息
然后调用handleBindApplication方法，方法内部会创建Instrumentation和App的Application然后  mInstrumentation.callApplicationOnCreate(app);回调Application.onCreate();

2.AMS调用thread.bindApplication后会继续往下走，继续调用mStackSupervisor.attachApplicationLocked(app)，方法内部调用thread.scheduleLaunchActivity方法，threa同样是ApplicationThread的Binder对象，所以继续看ApplicationThread.scheduleLaunchActivity, 方法内部也是通过handler发送LAUNCH_ACTIVITY消息，最终调用到Activity.handleLaunchActivity(r, null);（AMS通知ActivityThread调用handleLaunchActivity方式在不同系统版本上实现不一样。）
然后Instrumentation完成activity的创建和调用生命周期
 `activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);`

    if (r.isPersistable()) {
	    //调用Activity的OnCreate生命周期方法
	    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
    } else {
    	mInstrumentation.callActivityOnCreate(activity, r.state);
    }
这里面就看到了对Activity生命周期方法的调用了 Instrumentation.callActivityOnCreate

	public void callActivityOnCreate(Activity activity, Bundle icicle,
            PersistableBundle persistentState) {
        prePerformCreate(activity);
        activity.performCreate(icicle, persistentState);
        postPerformCreate(activity);
    }
Activity.performCreate

	final void performCreate(Bundle icicle) {
        onCreate(icicle);
        mActivityTransitionState.readState(icicle);
        performCreateCommon();
    }
后续的onStart和onResume也会在后面调用（不同系统版本实现不一样），到这里就从手机开机，到从Launcher开启第一个App的流程就到此结束。