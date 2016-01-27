---
title: SystemServer中的WatchDog
date: 2016-01-27 19:58:58
tags: Android系统
---
在***SystemServer***中创建***WatchDog***对象的代码如下:
```java
...
final Watchdog watchdog = Watchdog.getInstance();
watchdog.init(context, mActivityManagerService);
...
```
***Watchdog***是单实例的运行模式。第一次调用`getInstance()`会生成***Watchdog***的对象并保存到全局变量***sWatchdog***，再次调用时返回的保存的值。***WatchDog***的构造方法如下：
<!--more-->
```java
private Watchdog() {
    super("watchdog");
    mMonitorChecker = new HandlerChecker(FgThread.getHandler(),
            "foreground thread", DEFAULT_TIMEOUT);
    mHandlerCheckers.add(mMonitorChecker);
    mHandlerCheckers.add(newHandlerChecker(new Handler(Looper.getMainLooper()),
                "main thread", DEFAULT_TIMEOUT));
    mHandlerCheckers.add(new HandlerChecker(UiThread.getHandler(),
                "ui thread", DEFAULT_TIMEOUT));
    mHandlerCheckers.add(new HandlerChecker(IoThread.getHandler(),
                "i/o thread", DEFAULT_TIMEOUT));
    mHandlerCheckers.add(new HandlerChecker(DisplayThread.getHandler(),
                "display thread", DEFAULT_TIMEOUT));
}
```
***Watchdog***的构造方法的主要工作是创建了几个***HandlerChecker***对象，并把它们保存到数组列表***mHandlerCheckers***中。每个***HandlerChecker***对象对应一个被监控的线程。***HandlerChecker***类从***Handler***类派生，它在构造时就和被监控的线程关联到一起了。
除了在***Watchdog***的构造方法中构建的***HandlerChecker***对象外，还可以通过调用***Watchdog***类的`addThread()`方法来增加被监控的线程。***WatchDog***对象创建后，接下来会调用`init()`方法进行初始化，方法的代码如下：
```java
public void init(Context context, ActivityManagerService activity) {
    mResolver = context.getContentResolver();
    mActivity = activity;
    context.registerReceiver(new RebootRequestReceiver(),
            new IntentFilter(Intent.ACTION_REBOOT),
            android.Manifest.permission.REBOOT, null);
}
```
`init()`方法注册了***RebootRequestReceive***，监听重启的***intent：ACTION_REBOOT***。
***Watchdog***主要监控线程。在***SystemServer***进程中运行着许多的线程，它们负责处理一些重要模块的消息。如果一个线程陷入了死循环或者和其他线程相互死锁了，***Watchdog***需要有办法识别出它们。***Watchdog***中提供的两个方法`addThread()`和`addMonitor()`分别用来增加需要监控的线程和服务。`addThread()`实际是在创建一个和受监控对象关联的***HandlerChecker***对象，代码如下：
```java
public void addThread(Handler thread, long timeoutMillis) {
   synchronized (this) {
        if (isAlive()) {
            throw new RuntimeException("Threads can't be added once the Watchdog is running");
        }
        final String name = thread.getLooper().getThread().getName();
        mHandlerCheckers.add(new HandlerChecker(thread, name, timeoutMillis));
    } 
}
```
对服务的监控也是由***HandlerChecker***对象来完成。一个***HandlerChecker***对象就可以检查所有服务。因此，***Watchdog让mMonitorChecker***对象来完成这个任务。`addMonitor()`的代码如下：
```java
public void addMonitor(Monitor monitor) {
    synchronized (this) {
        if (isAlive()) {
            throw new RuntimeException("Monitors can't be added once the Watchdog is running");
        }
        mMonitorChecker.addMonitor(monitor);
    }
}
```
***Watchdog***的`addMonitor()`方法只是调用了***mMonitorChecker***对象的`addMonitor()`方法。在***Watchdog***的构造方法中，把5个公共线程加入到了监控列表中。
***(1) MainThread
(2) FgThread
(3) UiThread
(4) IoThread
(5) DisplayThread***
***SystemServer***中一些重要的服务拥有专用的线程来处理消息。同时***SystemServer***也启动了几个线程来为所有服务处理消息。这三个线程并没有太大区别，只是***UiThread***线程的优先级是***THREAD_PRIORITY_FOREGROUND***，而***FgThread***和***IoThread***线程的优先级是***THREAD_PRORITY_DEFAULT***而已。除了这5个公共线程外，一些重要服务的专用线程也加入到了监控中，包括:
***(1) ActivitiyManagerService的AThread线程。
(2) PackageManagerService的mHandlerThread变量表示的线程。
(3) PowerManagerService的线程。
(4) WindowManagerService的wmHandlerThread变量表示的线程。***
如果一个服务需要通过Watchdog来监控，它必须首先要实现***Watchdog***的接口***Monitor***：
```java
public interface Monitor {
    void monitor();
}
```
然后还要调用***Watchdog***类的`addMonitor()`方法把它自己加入到***Watchdog***的服务监控列表中，在***SystemServer***中实现了***Monitor***接口并调用了`addMonitor()`方法的服务有：
***(1) ActivityManagerService
(2) InputManagerService
(3) MediaRouterService
(4) MountService
(5) NativeDaemonConnector
(6) NetworkManagementService
(7) PowerManagerService
(8) WindowManagerService***
运行在***Binder***线程中的方法如果需要使用了全局的资源，就必须建立临界区来实施保护。通常的做法是使用***synchronized***关键字。通过给线程发送消息可以判断一个线程是否运行正常，如果发送的消息不能在规定的时间内得到处理，就表明线程被不正常占用了。Watchdog运行在一个单独的线程中，它的线程执行方法run()的代码如下：
```java
public void run() {
    boolean waitedHalf = false;
    while (true) {
        final ArrayList<HandlerChecker> blockedCheckers;
        final String subject;
        final boolean allowRestart;
        int debuggerWasConnected = 0;
        synchronized (this) {
            long timeout = CHECK_INTERVAL;
			//给监控的线程发送消息
            for (int i=0; i<mHandlerCheckers.size(); i++) {
                HandlerChecker hc = mHandlerCheckers.get(i);
                hc.scheduleCheckLocked();
            }
			//休眠一段时间
            if (debuggerWasConnected > 0) {
                debuggerWasConnected--;
            }
            long start = SystemClock.uptimeMillis();
            while (timeout > 0) {
                if (Debug.isDebuggerConnected()) {
                    debuggerWasConnected = 2;
                }
                try {
                    wait(timeout);
                } catch (InterruptedException e) {
                    Log.wtf(TAG, e);
                }
                if (Debug.isDebuggerConnected()) {
                    debuggerWasConnected = 2;
                }
                timeout = CHECK_INTERVAL - (SystemClock.uptimeMillis() - start);
            }
			//检查是否有线程或服务出现问题了
            final int waitState = evaluateCheckerCompletionLocked();
            if (waitState == COMPLETED) {
                waitedHalf = false;
                continue;
            } else if (waitState == WAITING) {
                continue;
            } else if (waitState == WAITED_HALF) {
                if (!waitedHalf) {
                    ArrayList<Integer> pids = new ArrayList<Integer>();
                    pids.add(Process.myPid());
                    ActivityManagerService.dumpStackTraces(true, pids, null, null, NATIVE_STACKS_OF_INTEREST);
                    waitedHalf = true;
                }
                continue;
                }
         ...
		{
			//杀死SystemServer
            Process.killProcess(Process.myPid());
            System.exit(10);
        }
        waitedHalf = false;
    }
}
```
`run()`方法中有一个无限循环，每次循环中主要做三件事。
(1) 调用`scheduleCheckLocked()`方法给所有受监控的线程发送消息。`shceduleCheckLocked()`方法的代码如下：
```java
public void scheduleCheckLocked() {
    if (mMonitors.size() == 0 && mHandler.getLooper().isIdling()) {
        mCompleted = true;
        return;
    }
    if (!mCompleted) {
        return;
    }
    mCompleted = false;
    mCurrentMonitor = null;
    mStartTime = SystemClock.uptimeMillis();
    mHandler.postAtFrontOfQueue(this);
}
```
***HandlerChecker***对象既要监控服务，又要监控线程，因此，上面的代码先判断***mMonitors***的***size***是否为0。如果为0，说明这个***HandlerChecker***没有监控服务，这是如果被监控的消息队列处于空闲状态，说明线程运行良好，把***mComplete***设为***true***后就可以返回了。否则先把***mComplete***设为***false***，然后纪录消息开始发送的时间到变量***mStartTime***中，最后调用`postAtFrontOfQueue()`方法给被监控的线程发送一个消息，这个消息处理的方法是***HandlerChecker***类的`run()`，代码如下：
```java
public void run() {
    final int size = mMonitors.size();
    for (int i = 0 ; i < size ; i++) {
        synchronized (Watchdog.this) {
            mCurrentMonitor = mMonitors.get(i);
        }
        mCurrentMonitor.monitor();
    }
    synchronized (Watchdog.this) {
        mCompleted = true;
        mCurrentMonitor = null;
    }
}
```
如果消息处理方法`run()`能够执行，说明受监控的线程本身没有问题，但是还需要检查被监控服务的状态。检查是通过调用服务中实现的`monitor()`方法来完成的。通常`monitor()`方法的实现是获取系统服务的锁，如果不能得到，线程就会挂起，这样***mComplete***的值就不能置为***true***了。
	***mComplete***的值为***true***，表明***HandlerChecker***对象监控的线程或服务正常，否则就可能会有问题。是有真有问题还要通过等待的时间是否超过规定时间来判断。
(2) 给受监控的线程发送完消息后，调用`wait()`方法让***watchdog***线程休眠一段时间。
(3) 逐个检查是否有线程或服务出现问题了，一旦发现问题，马上杀死进程。
前面的代码调用了方法`evaluateCheckerCompletionLocked()`来检查线程或服务是否有问题。`evaluateCheckerCompletionLocket()`方法的代码如下：
```java
private int evaluateCheckerCompletionLocked() {
    int state = COMPLETED;
    for (int i=0; i<mHandlerCheckers.size(); i++) {
        HandlerChecker hc = mHandlerCheckers.get(i);
        state = Math.max(state, hc.getCompletionStateLocked());
    }
    return state;
}
```
`evaluateCheckerCompletionLocked()`调用每个检查对象的`getCompletionStateLocked()`方法来得到对象的状态。状态值有4种。
**(1) COMPLETED**：值为0，表示状态良好。
**(2) WAITING**：值为1，表示正在等待消息处理的结果。
**(3) WAITED_HALF**：值为2，表示正在等待并且等待的时间超过规定时间的一半。
**(4) OVERDUE**：值为3，表示等待时间已经超过规定时间。
`evaluateCheckerCompletionLocked()`方法希望知道的是最坏的情况，因此，上面的代码使用***Math.max***来计算出所有监控对象的最坏情况。从上面的代码种可以看出返回的状态值只要不是***OVERDUE***都可以继续执行，否则就杀死***System***进程。
其实使用3种状态就足够了：***COMPLETED、WAITING、和OVERDUE***。***WAIT_HALF***这种状态是为了性调试。如果某个线程或服务的执行时间超过了规定时间的一半，表明可能会出现问题，系统就会把它的信息输出到Log中。
`getCompletionStateLocked()`函数用来返回***HandlerChecker***对象的状态，代码如下：
```java
public int getCompletionStateLocked() {
    if (mCompleted) {
        return COMPLETED;
    } else {
        long latency = SystemClock.uptimeMillis() - mStartTime;
        if (latency < mWaitMax/2) {
            return WAITING;
        } else if (latency < mWaitMax) {
            return WAITED_HALF;
        }
    }
    return OVERDUE;
}
```
从`getCompletionStateLocked()`函数的代码中我们可以看出，它正是通过时间来判断被监控对象的状况。
