---
title: SystemServer的创建过程
date: 2016-01-20 20:34:28
tags: Android系统
---
***SystemServer***的创建过程可以分成两个部分，一部分是在***Zygote***进程中***fork***并初始化***SystemServer***进程，另一部分是执行***SystemServer***类的***main***方法来启动系统服务。***init.rc***文件中定义的***Zygote***进程的启动参数包括了`—start-system-server`,因此，在ZygoteInit类的main方法里会调用`startSystemServer()`方法来启动SystemServer。startSystemServer()方法的代码如下：
<!--more-->
```java
private static boolean startSystemServer(String abiList, String socketName)
            throws MethodAndArgsCaller, RuntimeException {
        ...
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
        ZygoteConnection.Arguments parsedArgs = null;
        int pid;
        try {
            parsedArgs = new ZygoteConnection.Arguments(args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);

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
```
在`SystemServer()`方法中，主要做了三件事，第一件是为了***SystemServer***准备启动参数，从参数可以看到System进程的***PID***和***GID***都被指定为***1000***，SystemServer的执行类是`com.android.server.SystemServer`。
第二件事是调用***Zygote***类的`forkSystemServer()`来fork出SystemServer子进程。在***forkSystemServer***方法中调用了native层的函数来完成工作，native层的函数如下：
```cpp
static jint com_android_internal_os_Zygote_nativeForkSystemServer(
        JNIEnv* env, jclass, uid_t uid, gid_t gid, jintArray gids,
        jint debug_flags, jobjectArray rlimits, jlong permittedCapabilities,
        jlong effectiveCapabilities) {
  pid_t pid = forkAndSpecializeCommon(env, uid, gid, gids,
                                      debug_flags, rlimits,
                                      permittedCapabilities, effectiveCapabilities,
                                      MOUNT_EXTERNAL_NONE, NULL, NULL, true, NULL,
                                      NULL, NULL);
  if (pid > 0) {
      gSystemServerPid = pid;
      int status;
      if (waitpid(pid, &status, WNOHANG) == pid) {
          RuntimeAbort(env); 
      }
  }
  return pid;
}
```
在上面的代码中使用的`forkAndSpecializeCommon()`函数中将调用fork函数来创建子进程，在这之前，还调用了`SetSigChldHandler()`函数设置处理***SIGCHILD***信号的函数`SigChldHandler()`，代码如下：
```cpp
static void SigChldHandler(int /*signal_number*/) {
    pid_t pid;
    int status;
    while ((pid = waitpid(-1, &status, WNOHANG)) > 0) {
        if (WIFEXITED(status)) {
            if (WEXITSTATUS(status)) {
                ALOGI("Process %d exited cleanly (%d)", pid, WEXITSTATUS(status));
            }
        } else if (WIFSIGNALED(status)) {
            if (WTERMSIG(status) != SIGKILL) {
                ALOGI("Process %d exited due to signal (%d)", pid, WTERMSIG(status));
            }
            if (WCOREDUMP(status)) {
                ALOGI("Process %d dumped core.", pid);
            }
        }
        if (pid == gSystemServerPid) {
            ALOGE("Exit zygote because system server (%d) has terminated");
            kill(getpid(), SIGKILL);
        }
    }
  // Note that we shouldn't consider ECHILD an error because
  // the secondary zygote might have no children left to wait for.
  if (pid < 0 && errno != ECHILD) {
    ALOGW("Zygote SIGCHLD error in waitpid: %s", strerror(errno));
  }
}
```
`SigChldHandler()`函数接收到子进程死亡的信号后，除了调用`waitpid()`来防止子进程变”僵尸”外，还会判断死亡的子进程是否是***SystemServer***进程，如果是，***Zygote***进程会”自杀”，这样将导致***init***进程杀死所有用户进程并重启。
在fork出***SystemServer***后，在fork出的进程中调用`handleSystemServerProcess()`来初始化SystemServer进程。它的代码如下：
```java
private static void handleSystemServerProcess(
            ZygoteConnection.Arguments parsedArgs)
            throws ZygoteInit.MethodAndArgsCaller {
    closeServerSocket();
    Os.umask(S_IRWXG | S_IRWXO);
    if (parsedArgs.niceName != null) {
        Process.setArgV0(parsedArgs.niceName);
    }
    final String systemServerClasspath = Os.getenv("SYSTEMSERVERCLASSPATH");
    if (systemServerClasspath != null) {
        performSystemServerDexOpt(systemServerClasspath);
    }
    if (parsedArgs.invokeWith != null) {
        String[] args = parsedArgs.remainingArgs;
        if (systemServerClasspath != null) {
            String[] amendedArgs = new String[args.length + 2];
            amendedArgs[0] = "-cp";
            amendedArgs[1] = systemServerClasspath;
            System.arraycopy(parsedArgs.remainingArgs, 0, amendedArgs, 2,parsedArgs.remainingArgs.length);
        }
        WrapperInit.execApplication(parsedArgs.invokeWith,parsedArgs.niceName,parsedArgs.targetSdkVersion,null, args);
    } else {
        ClassLoader cl = null;
        if (systemServerClasspath != null) {
            cl = new PathClassLoader(systemServerClasspath, ClassLoader.getSystemClassLoader());
            Thread.currentThread().setContextClassLoader(cl);
        }
        RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
    }
    /* should never reach here */
}
```
`handleSystemServerProcess()`方法里首先关闭了从***Zygote***继承来的***socket***，接着将***SystemServer***进程的***umask***设为***0077(S_IRWXG|S_IRWXO)***，这样SystemServer创建的文件的属性就是0700，只有SystemServer进程可以访问。同时进程的名称修改为***system_server***。因为参数***invokeWith***通常为***null***，所以接下来还是通过***RuntimeInit***类的`zygoteInit()`方法来调用***SystemServer***类的`main()`方法。
***SystemServer***是Java类，它的入口`main()`方法的代码如下：
```java
public static void main(String[] args) {
    new SystemServer().run();
}
```
`main()`方法中创建了一个***SystemServer***的对象并调用`run()`函数，函数代码如下：
```java
private void run() {
    if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
        SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
    }
    // Here we go!
    Slog.i(TAG, "Entered the Android system server!");
    EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_SYSTEM_RUN, SystemClock.uptimeMillis());
    SystemProperties.set("persist.sys.dalvik.vm.lib.2", VMRuntime.getRuntime().vmLibrary());
    if (SamplingProfilerIntegration.isEnabled()) {
        SamplingProfilerIntegration.start();
        mProfilerSnapshotTimer = new Timer();
        mProfilerSnapshotTimer.schedule(new TimerTask() {
            @Override
            public void run() {
                SamplingProfilerIntegration.writeSnapshot("system_server", null);
            }
        }, SNAPSHOT_INTERVAL, SNAPSHOT_INTERVAL);
    }
    VMRuntime.getRuntime().clearGrowthLimit();
    VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);
    Build.ensureFingerprintProperty();
    Environment.setUserRequired(true);
    BinderInternal.disableBackgroundScheduling(true);
    android.os.Process.setThreadPriority(
            android.os.Process.THREAD_PRIORITY_FOREGROUND);
    android.os.Process.setCanSelfBackground(false);
    Looper.prepareMainLooper();
    System.loadLibrary("android_servers");
    nativeInit();
    performPendingShutdown();
    createSystemContext();
    mSystemServiceManager = new SystemServiceManager(mSystemContext);
    LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
    try {
        startBootstrapServices();
        startCoreServices();
        startOtherServices();
    } catch (Throwable ex) {
        throw ex;
    }
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```
`run()`方法的主要工作是：
（1） 调整时间。
（2） 设置属性***persists.sys.dalvik.vm.lib.2***的值为当前虚拟机的运行库路径。
（3） 调整虚拟机的内存。
（4） 装载库***libandroid_servers.so***。
（5） 调用`nativeInit()`方法初始化native层的***Binder服务***。
（6） 调用`createSystemContext()`来获取***Context***。
（7） 创建***SystemServiceManager***的对象***mSystemServiceManager***。
（8） 调用`startBootstrapServices()`方法，`startCoreServices()`方法和`startOtherServices()`创建并运行所有Java服务。
（9） 调用`Loop.loop()`，进入处理消息循环。
`nativeInit()`方法对应的native层函数`android_server_SystemServer_nativeInit()`，代码如下：
```cpp
static void android_server_SystemServer_nativeInit(JNIEnv* env, jobject clazz) {
    char propBuf[PROPERTY_VALUE_MAX];
    property_get("system_init.startsensorservice", propBuf, "1");
    if (strcmp(propBuf, "1") == 0) {
        SensorService::instantiate();
    }
}
```
从代码中可以看出，如果***system_init.startsensorservice***的值为1，则启动***SensorService***服务。***SensorService***提供的是各种传感器的服务，它也是***SystemServer***中唯一的本地服务。
在`run()`方法中调用的`createSystemContext()`方法，具体的代码如下：
```java
private void createSystemContext() {
     ActivityThread activityThread = ActivityThread.systemMain();
     mSystemContext = activityThread.getSystemContext();
     mSystemContext.setTheme(android.R.style.Theme_DeviceDefault_Light_DarkActionBar);
}
```
`createSystemContext()`方法调用***ActivityThread***的静态方法`systemMain()`来得到一个***ActivityThread***对象，让后调用它的`getSystemContext()`方法来得到系统的***Context***，最后设置主题。
SystemMain()方法的代码如下：
```java
public static ActivityThread systemMain() {
    if (!ActivityManager.isHighEndGfx()) {
        HardwareRenderer.disable(true); //关闭硬件渲染
    } else {
        HardwareRenderer.enableForegroundTrimming();
    }
    ActivityThread thread = new ActivityThread();
    thread.attach(true);
    return thread;
}
```
从代码中可以看出，`systemMain()`方法中创建了一个Activity对象。***ActivityThread***是应用程序的主线程类。***SystemServer***不仅仅是一个单纯的后台进程，它也是一个运行着组件***Service***的进程，很多系统的对话框就是从SystemServer中显示出来的，因此，SystemServer本身也需要一个和APK应用类似的上下文环境。`attach()`方法的代码如下：
```java
private void attach(boolean system) {
    sCurrentActivityThread = this;
    mSystemThread = system;
    if (!system) {
        ...
    } else {
        // Don't set application object here -- if the system crashes,
        // we can't display an alert, we just want to die die die.
        android.ddm.DdmHandleAppName.setAppName("system_process",
                UserHandle.myUserId());
        try {
            mInstrumentation = new Instrumentation();
            ContextImpl context = ContextImpl.createAppContext(
                this, getSystemContext().mPackageInfo);
            mInitialApplication = context.mPackageInfo.makeApplication(true, null);
            mInitialApplication.onCreate();
        } catch (Exception e) {
            throw new RuntimeException(
                "Unable to instantiate Application():" + e.toString(), e);
        }
    }
    ...
}
```
`attach()`方法在参数system为true时会创建一个类似应用的环境。这里创建了***ContextImpl***对象和***Application***对象，最后还调用了***Application***对象里的`onCreate()`方法，完全是在模拟创建一个应用。每一个上下文环境都需要对应一个APK文件，上面代码中创建的***contextImpl***对象使用的***PackageInfo***对象是通过`getSystemContext()`方法得到的，方法的代码如下：
```java
public ContextImpl getSystemContext() {
    synchronized (this) {
        if (mSystemContext == null) {
            mSystemContext = ContextImpl.createSystemContext(this);
        }
        return mSystemContext;
    }
}
```
`getSystemContext()`方法在***mSystemContext***为***null***的情况下会调用***createSystemContext***方法来创建一个***ContextImpl***对象，代码如下：
```java
static ContextImpl createSystemContext(ActivityThread mainThread) {
    LoadedApk packageInfo = new LoadedApk(mainThread);
    ContextImpl context = new ContextImpl(null, mainThread,
                     packageInfo, null, null, false, null, null);
    context.mResources.updateConfiguration(
    context.mResourcesManager.getConfiguration(),
    context.mResourcesManager.getDisplayMetricsLocked(Display.DEFAULT_DISPLAY));
    return context;
}
```
`createSystemContext()`方法中创建了一个***LoadedApk***对象，参数是***ActivityThread***。***LoadedApk***的构造方法如下：
```java
LoadedApk(ActivityThread activityThread) {
    mActivityThread = activityThread;
    mApplicationInfo = new ApplicationInfo();
    mApplicationInfo.packageName = "android";
    mPackageName = "android";
    ...
}
```
***LoadedApk***对象用来保存一个apk文件的信息，这个构造方法中会将使用的包名指定为***android***，而***framework-res.apk***的包名正是***android***。因此，`getSystemServer()`方法返回的是***mSystemContext***对象所对应的apk文件其实就是***framework-res.apk***。那么回到`attach()`方法中，新创建的***ContextImpl***对象实际上是在复制***mSystemContext***对象，接下来创建的Application对象实际代表了framework-res.apk。