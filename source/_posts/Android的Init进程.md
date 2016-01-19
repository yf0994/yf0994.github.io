---
title: Android的Init进程(一)
date: 2016-01-19 18:22:32
tags: Android系统
---
Init进程是Linux内核启动后创建的第一个用户进程，地位非常重要。Init进程在初始化过程中会启动很多重要的守护进程，因此，了解Init进程的启动过程将有助于我们更好的理解Android系统。Init进程除了完成系统的初始化之外，本身也是一个守护进程，担负着系统部分很重要的职责。
Android的启动过程可分为bootloader引导、装载和启动Linux内核、启动Android系统3个大的阶段。其中Android系统启动还可以细分为启动***Init进程、启动Zygote进程、启动SystemService、启动SystemServer、启动Home***等多个阶段。
下面简单介绍设备的启动过程。
### (1) Bootloader引导
当我们按下手机的电源键时，最先运行的就是***bootloader***。Bootloader主要的作用就是初始化基本的硬件设备（如CPU、内存、Flash等）并且通过建立内存空间映射，为装载Linux内核准备好合适的运行环境。一旦Linux内核装载完毕，bootloader将会从内存中清除掉。
如果用户在Bootloader运行期间，按下预定义的组合键，可以进入系统的更新模块。Android的下载更新可以选择进入Fastboot模式或者Recovery模式。
Fastboot是Android设计的一套通过USB来更新手机分区映像的协议，方便开发人员能快速更新指定的手机分区。
Recovery模式是Android特有的升级系统。利用Recovery模式，手机可以进行恢复出场设置或者执行OTA、补丁和固件升级。进入Recovery模式实际上是启动了一个文本模式的Linux。
### (2) 装载和启动Linux内核
Android的***boot.img***存放的就是Linux内核和一个根文件系统。Bootloader会把***boot.img***映像装载到内存。然后Linux内核会执行整个系统的初始化，完成后装载根文件系统，最后启动***Init进程***。
### (3) 启动Init进程
Linux内核加载完毕后，会首先启动Init进程，Init进程是系统的第一个进程。在Init进程的启动过程中，会解析Linux的配置脚本***init.rc***脚本。根据***init.rc***文件的内容，Init进程会***装载Android的文件系统、创建系统目录、初始化系统属性、启动Android系统重要的守护进程***，这些进程包括***USB守护进程、adb守护进程、vold守护进程、rild守护进程等***。最后Init进程也会作为守护进程来执行修改属性请求，重启崩溃的进程等作。
### (4) 启动ServiceManager
***ServiceManager***由Init进程启动。它的主要作用是管理***Binder服务***，负责Binder服务的注册与查找。
### (5) 启动Zygote进程
Init进程初始化结束时，会启动***Zygote进程***。Zygote进程负责fork出应用进程，是***所有应用进程的父进程***。Zygote进程初始化时会***创建Art虚拟机、预装载系统的资源文件和Java类***。所有从Zygote进程fork出的用户进程将继承和共享这些预加载的资源，不用浪费事件重新加载，加快了应用程序的启动过程。启动结束后，Zygote也将变为守护进程，负责响应启动APK应用程序的请求。
### (6) 启动SystemServer
***SystemServer***是Zygote进程fork出的第一个进程，也是整个Android系统的核心进程。在SystemServer中运行着Android系统大部分的Binder服务。SystemServer首先***启动本地服务SensorSerrvice***；接着启动包括***ActivityManagerService、WindowManagerService、PackageManagerService***在内的所有Java服务。
### (7) 启动MediaServer
***MediaServer***是由Init进程启动。它包含了一些多媒体相关的本地Binder服务，包括：***CameraService、AudioFlingerService、MediaPlayerService和PackageManagerService***在内的所有Java服务。
### (8) 启动Launcher
SystemServer加载完所有Java服务后，最后会调用`ActivityManagerService的SystemReady()方法`。在这个方法的执行中，会发出Intent `"android.intent.category.HOME"`。凡是响应这个Intent的APK应用都会运行起来，***Launcher***应用是Android系统默认的桌面应用，一般只有它会响应这个Intent，因此，系统开机后，第一个运行的应用就是Launcher。