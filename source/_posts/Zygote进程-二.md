---
title: Zygote进程(二)
date: 2016-01-26 20:04:24
tags: Android系统
---
***ZygoteInit***类负责***Zygote***进程Java层的初始化工作，入口方法`main()`去掉一些***log***后，具体的代码如下：
```java
public static void main(String argv[]) {
    try {
        SamplingProfilerIntegration.start();
        boolean startSystemServer = false;
        String socketName = "zygote";
        String abiList = null;
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
        if (abiList == null) {
            throw new RuntimeException("No ABI list supplied.");
        }
        registerZygoteSocket(socketName);
        EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
            SystemClock.uptimeMillis());
        preload();
        EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
            SystemClock.uptimeMillis());
        gc();
        Trace.setTracingEnabled(false);
        if (startSystemServer) {
            startSystemServer(abiList, socketName);
        }
        runSelectLoop(abiList);
        closeServerSocket();
    } catch (MethodAndArgsCaller caller) {
        caller.run();
    } catch (RuntimeException ex) {
       closeServerSocket();
       throw ex;
       }
}
```
<!--more-->
`main()`方法的主要工作是：
(1) 解析调用的参数。
(2) 调用`registerZygoteSocket()`方法注册***Zygote***的***socket***监听端口，用来接收启动应用程序的消息。
调用`preload()`方法装载系统资源，包括系统预加载类，***framework***资源和****OPENGL***的资源，这样当应用程序被fork处理后，应用的进程内已经包含了这些系统资源，大大节省了应用的启动时间。
(3) 调用`startSystemServer()`方法启动***SystemServer***进程。
(4) 最后调用`runSelectLoop()`方法进入监听和接收消息的循环。
***ZygoteInit***类的`main()`方法首先调用`registerZygoteSocket()`方法来创建一个本地socket，接着调用`runSelectLoop()`来进入等待socket连接的循环中。
`registerZygoteSocket()`方法的代码如下：
```java
private static void registerZygoteSocket(String socketName) {
    if (sServerSocket == null) {
        int fileDesc;
        final String fullSocketName = ANDROID_SOCKET_PREFIX + socketName;
        try {
            String env = System.getenv(fullSocketName);
            fileDesc = Integer.parseInt(env);
        } catch (RuntimeException ex) {
            throw new RuntimeException(fullSocketName + " unset or invalid", ex);
        }
        try {
            sServerSocket = new LocalServerSocket(
                    createFileDescriptor(fileDesc));
        } catch (IOException ex) {
            throw new RuntimeException(
                    "Error binding to local socket '" + fileDesc + "'", ex);
        }
    }
}
```
`registerZygoteSocket()`方法通过环境变量***ANDROID_SOCKET_zygote***来获取socket句柄。
***Init***进程会根据这条选项来创建一个***AF_UNIX***的***socket***，并把它的句柄放在环境变量***ANDROID_SOCKET_zygote***中。这个环境变量字符串后面部分的`zygote`就是选项的名称。得到句柄后通过`createFileDescirptor()`方法来生成一个***FileDescribator***的实例对象，然后再调用`LocalServerSocket()`来创建一个本地***socket***，它的值保存在全局变量***sServerSocket***中。
`createFileDescriptor()`方法是一个***native***方法，它在native层对应的函数会创建类***java.io.FileDescriptor***的对象，并把参数句柄值保存到***FileDescriptor***类的成员变量***descriptor***中。Android启动一个新的进程是在***ActivityManagerService***中完成的，可能会有多种原因导致系统启动一个新的进程，最终在***ActivityManagerService***中都是通过调用方法`startProcessLocked()`来完成这一过程的。
`startProcessLocked()`方法准备好参数后，调用***Process***类的静态方法`start()`来启动应用，代码如下：
```java
...
rocess.ProcessStartResult startResult = Process.start(entryPoint,
                                            app.processName, uid, uid, gids,
                                            debugFlags, mountExternal,
                                            app.info.targetSdkVersion,
                                            app.info.seinfo, requiredAbi,
                                            instructionSet,app.info.dataDir, entryPointArgs);
...
```
`start()`方法的第一个参数***android.app.ActivithTread***就是应用启动后执行的Java类。`Start()`方法通过调用`startViaZygote()`方法来启动应用。`startViaZygote()`方法首先将应用进程的启动参数保存到***argsForZygote***列表中，然后调用`zygoteSendArgsAndGetPid()`方法将应用程序进程的启动参数发送到***Zygote***进程，`startViaZygote()`方法的代码如下：
```java
private static ProcessStartResult startViaZygote(final String processClass,
                                  ... )
                                  throws ZygoteStartFailedEx {
    synchronized(Process.class) {
        ArrayList<String> argsForZygote = new ArrayList<String>();
        argsForZygote.add("--runtime-init");
        argsForZygote.add("--setuid=" + uid);
        argsForZygote.add("--setgid=" + gid);
        ...
        return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
    }
}
```
`zygoteSendArgsAndGetResult()`方法会把参数发送给***Zygote***进程，在这个方法中，会调用`openZygoteSocketIfNeed()`方法来创建用于通信的***socket***，具体如下所示：
```java
...
sZygoteSocket = new LocalSocket();
sZygoteSocket.connect(new LocalSocketAddress(ZYGOTE, LocalSocketAddress.Namespace.RESERVED));
...
```
宏***ZYGOTE_SOCKET***的定义是字符串***zygote***，对于***LocalSocket***，使用字符串作为地址就可以通信，非常方便。***ZygoteInit***类的`main()`方法会调用`selectReadable()`方法来检测是否有***socket***事件到来。检测到事件后，就会唤醒进程处理。代码如下：
```java
private static void runSelectLoop(String abiList) throws MethodAndArgsCaller {
    ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
    ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();
    FileDescriptor[] fdArray = new FileDescriptor[4];
    fds.add(sServerSocket.getFileDescriptor());
    peers.add(null);
    int loopCount = GC_LOOP_COUNT;
    while (true) {
		...
        try {
            fdArray = fds.toArray(fdArray);
            index = selectReadable(fdArray);
        } catch (IOException ex) {
            throw new RuntimeException("Error in select()", ex);
        }
        ...
        }
    }
}
```
如果`selectReadable()`方法返回的***index***值为0，说明是请求连接的事件到来了，这时调用`acceptCommandPeer()`来和客户端建立一个***socket***连接，然后把这个***socket***加入到监听数组中，等待这个***socket***上命令的到来，具体代码如下：
```java
...
if (index < 0) {
    throw new RuntimeException("Error in select()");
    } else if (index == 0) {
        ZygoteConnection newPeer = acceptCommandPeer(abiList);
        peers.add(newPeer);
        fds.add(newPeer.getFileDescriptor());
    } else {
        boolean done;
        done = peers.get(index).runOnce();
        if (done) {
            peers.remove(index);
            fds.remove(index);
        }
}
...
```
如果`selectReadable()`方法返回的***index***值大于0，说明是已经连接的***socket***上有数据到了，一旦接收到和客户端连接的socket传过来的命令，`runSelectLoop()`方法会调用***ZygoteConnection***类的`runOnce()`方法去处理命令，处理完毕后会断开与客户端的连接，并用于连接的socket从监听列表清除。
***ZygoteConnection***类的`runOnce()`方法负责创建子进程。`runOnce()`首先调用`readArgumentList()`方法从socket连接中读入多个参数行，参数行的样式是`—setuid=1`，行与行之间以`\r`、`\n`或者`\r\n`分割。读完后再调用***Argments***类的`parseArgs()`方法把它们解析成参数列表。
在解析完参数后，`runOnce()`方法还会对这些参数进行检查和设置，具体如下所示：
```java
...
applyUidSecurityPolicy(parsedArgs, peer, peerSecurityContext);
applyRlimitSecurityPolicy(parsedArgs, peer, peerSecurityContext);
applyInvokeWithSecurityPolicy(parsedArgs, peer, peerSecurityContext);
applyseInfoSecurityPolicy(parsedArgs, peer, peerSecurityContext);
...
```
其中`applyUidSecurityPolicy()`方法将检查客户端进程是否有权利指定进程的用户ID，组ID和所属的组。具体的规则是：如果客户端进程是***root***进程，则可以任意指定；如果客户端进程是***System***进程，只有在系统属性`ro.factorytest`的值为1或2的情况下可以指定；其余情况报错。如果没有指定用户ID和组ID，将继承客户端进程的值。
方法`applyRlimitSecurityPolicy()`、`applyInvokeWithSecurityPolicy()`和`applyseInfoSecurityPolicy()`主要是检查客户端是否有资格让zygote进程来执行相关的系统调用。这种检查依据的是SELinux定义的安全上下文的设置。参数检查无误后，`runOnce()`方法将调用***Zygote***类的`forkAndSpecialize()`方法fork子进程：
```java
...
pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid,
                                parsedArgs.gids,parsedArgs.debugFlags,
                                rlimits, parsedArgs.mountExternal,
                                parsedArgs.seInfo,parsedArgs.niceName,
                                fdsToClose, parsedArgs.instructionSet,
                                parsedArgs.appDataDir);
...
```
`forkAndSpecialize()`方法最终是通过native层的`forkAndSpecializeCommon()`函数来完成fork工作，其主要工作是：
(1) fork出子进程。
(2) 在子进程中挂载***external storage***。
(3) 在子进程中设置用户ID，组ID和进程所属的组。
(4) 在子进程中执行系统调用***setrlimit***来设置进程的系统资源限制。
(5) 在子进程中执行系统调用***capset***来设置进程的权限。
(6) 在子进程中设置应用进程的安全上下文。
(7) 如果使用的Dalvik虚拟机，还会调用***dvmInitAfterZygote()***来执行Dalvik虚拟机的一些初始化工作。
从native层返回Java层后，`runOnce()`方法会继续执行，如果返回的值pid等于0，则表示处于子进程中，执行`handleChildProc()`方法。如果pid不等于0，则表示在Zygote进程中，然后调用`handleParentProc()`方法继续执行。具体的代码如下所示：
```java
...
if (pid == 0) {
	//在子进程中
    IoUtils.closeQuietly(serverPipeFd);
    serverPipeFd = null;
    handleChildProc(parsedArgs, descriptors, childPipeFd, newStderr);
    return true;
} else {
	//在父进程中
    IoUtils.closeQuietly(childPipeFd);
    childPipeFd = null;
return handleParentProc(pid, descriptors, serverPipeFd, parsedArgs)
}
...
```
***Zygote***进程fork出子进程后，调用`handleChildProc()`方法来完成子进程的初始化工作。`handleChildProc()`方法首先关闭了监听的***socket***和从***Zygote***中继承来的文件描述符，具体代码如下：
```java
private void handleChildProc(Arguments parsedArgs,FileDescriptor[] descriptors, FileDescriptor pipeFd, PrintStream newStderr)throws ZygoteInit.MethodAndArgsCaller {
    closeSocket();
    ZygoteInit.closeServerSocket();
    if (descriptors != null) {
        try {
            ZygoteInit.reopenStdio(descriptors[0],
                    descriptors[1], descriptors[2]);
            for (FileDescriptor fd: descriptors) {
                IoUtils.closeQuietly(fd);
            }
            newStderr = System.err;
        } catch (IOException ex) {
            Log.e(TAG, "Error reopening stdio", ex);
        }
    }
    ...
}
```
接下来根据启动参数是否有`—runtime-init`以及`—invoke-with`来判断如何初始化。具体代码如下：
```java
...
if (parsedArgs.runtimeInit) {
    if (parsedArgs.invokeWith != null) {
        WrapperInit.execApplication(parsedArgs.invokeWith,
                parsedArgs.niceName, parsedArgs.targetSdkVersion,
                pipeFd, parsedArgs.remainingArgs);
    } else {
        RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion,
                parsedArgs.remainingArgs, null /* classLoader */);
    }
}
...
```
启动apk应用都会带有`—runteim-init`参数，但参数`invokeWith`的值通常为`NULL`，`invokeWith`不为`NULL`将会通过调用***exec***的方式启动***app_process***进程来执行Java类。正常情况下会调用***RuntimeInit***类的`zygoteInit()`方法，这个方法的代码如下：
```java
public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
    redirectLogStreams();
    commonInit();
    nativeZygoteInit();
    applicationInit(targetSdkVersion, argv, classLoader);
}
```
`zygoteInit()`方法又调用了3个方法`commonInit()`、`nativeZygoteInit()`、和`applicationInit()`来执行初始化，这3个方法的作用如下：
`commonInit()`方法执行的是一些简单的初始化，包括：
`Thread.setDefaultUncaughtExceptionHandler(new UncaughtHandler());`
对于应用中没有捕获处理的***Exception***，系统缺省的处理方法是打印出错的栈信息，然后退出进程。应用也可以调用`Thread.setDefaultUncaughtExceptionHandler()`方法来设置自己的处理方法，例如在自己的处理方法中把产生的异常信息发送到开发者的邮箱里，方便解决问题。
**设置时区：**
```java
...
TimezoneGetter.setInstance(new TimezoneGetter() {
    @Override
    public String getId() {
        return SystemProperties.get("persist.sys.timezone");
    }
});
TimeZone.setDefault(null);
...
```
**重置Android的Log系统以及设置属性`http.agent`：**
```java
...
LogManager.getLogManager().reset();
new AndroidConfig();
String userAgent = getDefaultUserAgent();
System.setProperty("http.agent", userAgent);
NetworkManagementSocketTagger.install();
...
```
`nativeZygote()`方法是本地方法，会调用到native层的函数，具体代码如下：
```cpp
...
static void com_android_internal_os_RuntimeInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    gCurRuntime->onZygoteInit();
}
...
```
`com_android_internal_os_RuntimeInit_nativeZygote()`函数只有一行，***gCurRuntime***是指向***AppRuntime***的全局指针，它的`onZygoteInit()`函数的代码如下所示：
```java
virtual void onZygoteInit()
{
    atrace_set_tracing_enabled(true);
    sp<ProcessState> proc = ProcessState::self();
    proc->startThreadPool();
}
```
`onZygoyrInit()`函数调用了`ProcessState::self()`函数和***ProcessState***类的`startThreadPool()`函数，这两条函数是用来初始化***Binder***的使用环境，这样应用进程就可以使用***Binder***了。
**applicationInit()方法**
在`applicationInit()`方法中，除了设置虚拟机的两个参数外，最重要的是调用了`invokeStaticMain()`方法来跳转到Java层执行：
```java
...
invokeStaticMain(args.startClass, args.startArgs, classLoader);
...
```
调用函数的参数***args.startClass***是通过***socket***传入的，通常是`android.app.ActivityThread`。这样`invokeStaticMain()`方法将会调用***ActivityThread***类的`main()`方法，但是在`invokeStaticMain()`方法中并不是直接调用`main()`方法，而是抛出了一个异常：
```java
...
throw new ZygoteInit.MethodAndArgsCaller(m, argv);
...
```
在***ZygoteInit***的`main()`方法中会捕获这个异常:
```java
...
catch (MethodAndArgsCaller caller) {
caller.run();
}
...
```
在`caller.run()`中才会真正调用***ActivityThread***的`main()`方法，这样就进入应用本身的启动流程。