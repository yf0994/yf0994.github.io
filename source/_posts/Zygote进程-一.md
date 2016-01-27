---
title: Zygote进程(一)
date: 2016-01-26 19:13:28
tags: Android系统
---
***Zygote***进程是Android中非常重要的一个进程，它和***Init进程、SystemServer进程***是支撑Android世界的三极。Linux进程使用系统调用***fork***产生的，***fork***出的子进程除了内核中的一些核心数据结构和父进程不相同外，其余的内存映像都是和父进程相同的。只有当子进程需要去修改写这些共享内存时，操作系统才会为子进程分配一个新的页面，并将老的页面上的数据复制到一份新的页面，这就是所谓的***写时拷贝***。
通常子进程被***fork***出来后，会继续执行系统调用***exec***，***exec***将用一个新的可执行文件内容替换当前进程的代码段，数据段，堆和栈段。***Fork***加***exec***是Linux启动应用的标准做法，***Init***进程也是这样来启动其他服务的。
<!--more-->
***Zygote***创建应用程序时却只是用了***fork***，没有调用***exec***。Android应用程序中执行的是***Java***代码，Java代码的不同才造成了应用的区别，而对于运行Java环境，要求却是一样的。***Zygote***初始化会创建虚拟机，同时把需要的系统类库和资源文件加载到内存里。***Zygote***进程***fork***出子进程后，这个进程也继承了能正常工作的虚拟机和各类系统资源，接下来子进程只需要装载APK文件中的字节码就可以运行了。这样应用启动的时间会大大缩短。
***Zygote***进程在***Init***进程中以***Service***的形式来启动的，在***Android5.0***中，***Zygote***的启动发生了变化，以前直接放在***inir.rc***文件中的代码块放到了单独的文件中，在***init.rc***中则通过***import***的方式引入文件，如下所示：
`import /init.${ro.zygote}.rc`
从上面的语句可以看到，***init.rc***并不是直接引入某个固定的文件，而是根据属性***ro.zygote***的内容来引入不同的文件。这是因为从***Android5.0***开始，Android开始支持***64位***的编译，***Zygote***进程本身也会有32位和64位版本的区别。
***Zygote***进程的可执行文件是***app_process***。***App_process***模块的源码文件位于目录***framework/base/cmds/app_process***下，只有一个***app_process.cpp***文件。
***App_process***文件中的`main()`函数的主要功能是解析启动参数。虽然***Zygote***进程是通过可执行文件***app_process***创建的，但是，***app_process***除了能创建***Zygote***进程外，还能创建出普通进程。
***App_process***进程启动参数的格式是：
**app_process ［虚拟机参数］ 运行目录 参数［Java类］**
虚拟机参数：以符号‘－’开头。启动dalvik虚拟机时传递给虚拟机。
运行目录：程序的运行目录，通常是`/system/bin`。
参数：以符号`--`开头，参数`--zygote`表示要启动***zygote***进程。参数`--application`表示要以普通进程的方式执行Java代码。
Java类：要执行的Java类，必须有一个静态方法main。使用参数`--Zygote`时不会执行这个类，而是固定的执行***ZygoteInit***类。
理解了参数的区别之后，我们对`main()`函数进行分析。
**(1) 创建AppRuntime对象并保存参数**
AppRuntime是在app_process中定义的类，继承了AndroidRuntime类，AndroidRuntime类的主要作用是创建和初始化虚拟机：
```c
int main(int argc, char* const argv[])
{
    if (prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0) < 0) {
        if (errno != EINVAL) {
            LOG_ALWAYS_FATAL("PR_SET_NO_NEW_PRIVS failed: %s", strerror(errno));
            return 12;
        }
    }
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    // Process command line arguments
    // ignore argv[0]
    argc--;
    argv++;
    int i;
    for (i = 0; i < argc; i++) {
        if (argv[i][0] != '-') {
            break;
        }
        if (argv[i][1] == '-' && argv[i][2] == 0) {
            ++i; // Skip --.
            break;
        }
        runtime.addOption(strdup(argv[i]));
    }
    ...
}
```
**(2) 解析启动参数**
```c
...
while (i < argc) {
    const char* arg = argv[i++];
    if (strcmp(arg, "--zygote") == 0) {
        zygote = true;
        niceName = ZYGOTE_NICE_NAME;
    } else if (strcmp(arg, "--start-system-server") == 0) {
        startSystemServer = true;
    } else if (strcmp(arg, "--application") == 0) {
        application = true;
    } else if (strncmp(arg, "--nice-name=", 12) == 0) {
        niceName.setTo(arg + 12);
    } else if (strncmp(arg, "--", 2) != 0) {
        className.setTo(arg);
        break;
    } else {
        --i;
        break;
    }
}
...
```
通常***init.rc***中传入参数`-Xzygote /system/bin –zygote –start-system-server`
解析后得到的结构将是：
变量***parentDir***等于***/system/bin***。
变量***niceName***等于***zygote***。
变量***startSystemServer***等于***true***。
变量***zygote***等于***true***。
**(3) 准备执行ZygoteInit类或RuntimeInit类的参数：**
```c
...
Vector<String8> args;
    if (!className.isEmpty()) {
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);
    } else {
    // We're in zygote mode.
    maybeCreateDalvikCache();
    if (startSystemServer) {
        args.add(String8("start-system-server"));
    }
    char prop[PROP_VALUE_MAX];
    if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
        LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
            ABI_LIST_PROPERTY);
        return 11;
    }
    String8 abiFlag("--abi-list=");
    abiFlag.append(prop);
    args.add(abiFlag);
    for (; i < argc; ++i) {
        args.add(String8(argv[i]));
    }
}
...
```
**(4) 将本进程的名称改为参数“—nice-name”指定的字符串:**在缺省情况下niceName的值为“zygote”或“zygote64”。
```c
...
if (!niceName.isEmpty()) {
    runtime.setArgv0(niceName.string());
    set_process_name(niceName.string());
}
...
```
`set_process_name()`函数用来改变进程的名称。`setArgv0()`函数是用来替换启动参数串中的***app_process***为参数`—nice-name`指定的名称。
**(5) 启动Java类:**
如果启动参数带有“--zygote”，执行ZygoteInit类，否则执行通过参数传递进来的Java类：
```c
...
if (zygote) {
    runtime.start("com.android.internal.os.ZygoteInit", args);
} else if (className) {
    runtime.start("com.android.internal.os.RuntimeInit", args);
} else {
    fprintf(stderr, "Error: no class name or --zygote supplied.\n");
    app_usage();
    LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    return 10;
}
...
```
***app_process***除了能启动Zygote进程外，也可以使用它来执行某个系统的Java类。Android手机中常用的工具***am***就是一个很好的例子。***am***是一个通过发送***Intent***来启动应用程序的工具。但是***am***实际上只是一个包含几行代码的脚本文件，它的功能都是调用***app_process***来完成的。如下所示：
```shell
#! /system/bin/sh
base=/system
export CLASSPATH=$base/framework/am.jar
exec app_process $base/bin com.android.commands.am.Am “$@”
```
***AndroidRuntime***类是Android底层中很重要的一个类，它负责启动虚拟机以及Java线程。***AndroidRuntime***类在一个进程中只有一个实例对象，保存在全局变量***gCurRuntime***中。
AndroidRuntime类的构造函数如下：
```cpp
AndroidRuntime::AndroidRuntime(char* argBlockStart, const size_t argBlockLength) :
        mExitWithoutCleanup(false),
        mArgBlockStart(argBlockStart),
        mArgBlockLength(argBlockLength)
{
    SkGraphics::Init();  //初始化skia图形系统
    //预分配空间来存放传入虚拟机的参数
    mOptions.setCapacity(20);
    assert(gCurRuntime == NULL);//只能被初始化一次
    gCurRuntime = this;
}
```
在***AndroidRuntime***的构造函数中，首先对***skia***系统进行初始化，然后调用***mOptions.setCapacity(20)***来设置虚拟机参数。
最后***AndroidRuntime***对象的指针被放进了***gCurRuntime***全局指针。
在前面的`main()`函数结尾，调用了***AndroidRuntime***类的`start()`函数来执行Java类。***Zygote***进程运行Java代码前，还需要初始化整个Java运行环境，`start()`函数的执行过程如下：
**(1) 打印启动Log**
```cpp
ALOGD("\n>>>>>> AndroidRuntime START %s <<<<<<\n",
        className != NULL ? className : "(unknown)");
```
**(2) 获取系统目录**
系统目录是从环境变量***ANDROID_ROOT***读取。如果没有设置，则默认为目录***/system***，如果手机下没有***system***目录，***Zygote***进程会退出。系统目录是在***Init***进程中创建出来的。
```c
...
const char* rootDir = getenv("ANDROID_ROOT");
if (rootDir == NULL) {
    rootDir = "/system";
if (!hasDir("/system")) {
    return;
}
setenv("ANDROID_ROOT", rootDir, 1);
...
```
**(3) 启动虚拟机**
```cpp
...
JniInvocation jni_invocation;
jni_invocation.Init(NULL);
JNIEnv* env;
if (startVm(&mJavaVM, &env) != 0) {
    return;
}
...
```
从Android5.0开始，启动的将是ART虚拟机。
**(4) 调用onVmCreated(env)***
`onVmCreated()`是一个虚函数，调用它实际上调用的是继承类***AppRuntime***中的重载函数，下面是***AppRuntime***类的`onVmCreate()`函数代码：
```cpp
virtual void onVmCreated(JNIEnv* env){
    if (mClassName.isEmpty()) {
        return; // Zygote. Nothing to do here.
    }
    char* slashClassName = toSlashClassName(mClassName.string());
    mClass = env->FindClass(slashClassName);
    if (mClass == NULL) {
    }
    free(slashClassName);
    mClass = reinterpret_cast<jclass>(env->NewGlobalRef(mClass));
}
```
在***AppRuntime***中的`onVmCreate()`函数中，如果是***Zygote***进程，变量***mClassName***的值为***NULL***，立即就返回了。如果***app_process***是在执行对一个普通Java类的调用，***mClassName***变量的值将是类的名称。
`toSlashClassName()`函数的作用是把类名从格式`com.android.ZygoteInit`转化为格式`/com/android/ZygoteInit`。在Native层，表示一个Java类名是用后一种方式。
`OnVmCreate()`函数会在当前的虚拟机环境中根据类名查找类对象。这表明***app_process***将要调用的Java类必须是系统的Java类，而不能是任意应用中的Java类，那么普通应用中的Java类也能被调用执行吗，答案是肯定的，只是调用方式有些不同，稍微麻烦。
**(5) 注册系统的JNI函数：**
```cpp
...
if (startReg(env) < 0) {
    ALOGE("Unable to register all android natives\n");
    return;
}
...
```
**(6) 准备调用Java类的main函数的参数:**
```cpp
...
jclass stringClass;
jobjectArray strArray;
jstring classNameStr;
stringClass = env->FindClass("java/lang/String");
assert(stringClass != NULL);
strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
assert(strArray != NULL);
classNameStr = env->NewStringUTF(className);
assert(classNameStr != NULL);
env->SetObjectArrayElement(strArray, 0, classNameStr);
for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
}
...
```
上面代码中首先调用`NewObjectArray()`函数生成一个包含两个元素的数组对象***strArray***，接着调用`SetObjectArrayElement()`函数来为每个数组元素赋值。
**(7) 调用ZygoteInit类的main()方法:**
```cpp
...
char* slashClassName = toSlashClassName(className);
jclass startClass = env->FindClass(slashClassName);
if (startClass == NULL) {
} else {
    jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
        "([Ljava/lang/String;)V");
    if (startMeth == NULL) {
    } else {
        env->CallStaticVoidMethod(startClass, startMeth, strArray);
    }
}
...
```
调用***main()***方法之前，先通过函数`GetStaticMethodID()`获得了***main()***方法的ID，接下来就是调用`CallStaticVoidMethod()`函数来调用Java层的函数了。如果不是启动***Zygote***，执行Java类的将是***RuntimeInit***。