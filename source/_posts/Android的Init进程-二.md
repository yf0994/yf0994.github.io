---
title: Android的Init进程(二)
date: 2016-01-19 18:51:37
tags: Android系统
---
Init进程的源码位于目录***system/core/init***下。***程序的入口函数main()***位于文件init.c中。main()函数比较长，整个Init进程的启动流程都在这个函数中。下面我们把main()函数分成小段，一段段地介绍其功能和作用。
进入main()函数后，首先检查启动程序的文件名。如果文件名是***uevent***，执行守护进程的uevent的主函数***uevent_main()***，如果文件名是***watchdogd***，执行看门狗守护进程的主函数***watchdogd_main()***。若都不是则继续执行。
```c
int main(int argc, char **argv){
    ...
    
    if (!strcmp(basename(argv[0]), "ueventd"))
        return ueventd_main(argc, argv);

    if (!strcmp(basename(argv[0]), "watchdogd"))
        return watchdogd_main(argc, argv);
    
    /* clear the umask */
    umask(0);
    ...
}
```
从这里可以看出init进程的代码里也包含了另外两个守护进程的代码，因为这几个守护进程的代码重合度高，所以，开发人员干脆把它们都放在一起了。但是在编译时，Android生成了两个指向init文件的符号连接uevent和watchdogd，这样启动时如果执行的是这两个符号连接，main()函数就能判断出到底要启动哪个守护进程。
缺省情况下一个进程创建出的文件和文件夹的属性是***022***，使用***umask()***函数能设置文件属性的掩码。参数为0意味着进程创建的文件属性是***0777***。
接着创建一些基本的目录，包括/dev、/proc、/sys等；同时把一些文件系统，如tmpfs、devpt、proc、sysfs等mount到相应的目录。
```c
    ...
    /* Get the basic filesystem setup we need put
     * together in the initramdisk on / and then we'll
     * let the rc file figure out the rest.
    */
    mkdir("/dev", 0755);
    mkdir("/proc", 0755);
    mkdir("/sys", 0755);

    mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
    mkdir("/dev/pts", 0755);
    mkdir("/dev/socket", 0755);
    mount("devpts", "/dev/pts", "devpts", 0, NULL);
    mount("proc", "/proc", "proc", 0, NULL);
    mount("sysfs", "/sys", "sysfs", 0, NULL);
    
    /* indicate that booting is in progress to background fw loaders, etc */
    close(open("/dev/.booting", O_WRONLY | O_CREAT, 0000));
    ...
```
***tmpfs***是一种基于内存的文件系统，mount后就可以使用。tmpfs文件系统下的文件都存放在内存中，访问速度快，但是关机后所有内容都会丢失，因此tmpfs文件系统比较适合存放一些临时性文件。tmpfs文件系统的大小是动态变化的，刚开始占用空间很小，随着文件的增多会随着变大。Android将tmpfs文件系统mount到/dev目录，/dev目录用来***存放系统创建的设备节点***，正好符合tmpfs文件系统的特点。
***devpts***是虚拟终端文件系统，它通常mount在目录`/dev/pts`下。
***proc***也是一种***基于内存的虚拟文件系统***，它可以看作是内核内部数据结构的接口，通过它可以获得系统的信息，同时能够运行时修改特定的内核参数。
***sysfd***文件系统和proc文件系统类似，它是Linux2.6内核引入的，作用是把系统的设备和总线按层次组织起来，使得它们可以在用户空间存取，用来向用户空间导出内核的数据结构及它们的属性。
在***/dev***    目录下创建一个空文件***.booting***,表示初始化正在运行。***is_booting()***函数会依靠空文件***.booting***来判断是否进程处于初始化中。初始化结束后这个文件将被删除。
接下来的代码如下:
```c
    /* We must have some place other than / to create the
     * device nodes for kmsg and null, otherwise we won't
     * be able to remount / read-only later on.
     * Now that tmpfs is mounted on /dev, we can actually
     * talk to the outside world.
     */
    open_devnull_stdio();
    klog_init();
    property_init();
    get_hardware_name(hardware, &revision);
    process_kernel_cmdline();
```
调用***open_devnull_stdio()***函数把标准输入、标准输出和标准错误重定向到空设备文件***/dev/_null_***，这是创建守护进程常用的手段;调用***klog_init()***函数创建节点***/dev/__kmsg__***，这样init进程可以使用kernel的log系统来输出log了,这是因为这时Android的log系统还没有启动，所以init只能使用***kernel的log系统***;调用***property_init()***函数来初始化Android的属性系统,***property_init()***函数的主要作用是创建一个共享区域来存储属性值;调用***get_hardware_name()***函数通过分析***/proc/cpuinfo***文件，取得系统硬件的名称;调用***process_kernel_cmdline()***函数解析kernel的启动参数，启动参数通常放在***/proc/cmdline***中。解析的结果是利用参数中的值来设置几个属性值，包括:***ro.serialno***、***ro.bootmode***、***ro.baseband***、***ro.bootloader***。
接下来初始化***SELinux***,代码如下:
```c
    ...
    union selinux_callback cb;
    cb.func_log = log_callback;
    selinux_set_callback(SELINUX_CB_LOG, cb);

    cb.func_audit = audit_callback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);

    selinux_initialize();
    /* These directories were necessarily created before initial policy load
     * and therefore need their security context restored to the proper value.
     * This must happen before /dev is populated by ueventd.
     */
    restorecon("/dev");
    restorecon("/dev/socket");
    restorecon("/dev/__properties__");
    restorecon_recursive("/sys");
    
    is_charger = !strcmp(bootmode, "charger");
    property_load_boot_defaults();
    init_parse_config_file("/init.rc");
    ...
```
***property_load_boot_defaults()***函数将解析设备根目录下的***default.prop***文件，把这个文件中定义的属性值读出来设置到属性系统中,接着调用***init_parse_config_file()***函数解析***init.rc***文件,解析完成后的结果是将init文件中的Server项和Action项分别加入到内部Service列表***service_list***和Action列表***action_list***中。
```c
    ...
    action_for_each_trigger("early-init", action_add_queue_tail);
    queue_builtin_action(wait_for_coldboot_done_action, "wait_for_coldboot_done");
    queue_builtin_action(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
    queue_builtin_action(keychord_init_action, "keychord_init");
    queue_builtin_action(console_init_action, "console_init");

    /* execute all the boot actions to get us started */
    action_for_each_trigger("init", action_add_queue_tail);

    /* Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random
     * wasn't ready immediately after wait_for_coldboot_done
     */
    queue_builtin_action(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
    queue_builtin_action(property_service_init_action, "property_service_init");
    queue_builtin_action(signal_init_action, "signal_init");

    /* Don't mount filesystems or start core system services if in charger mode. */
    if (is_charger) {
        action_for_each_trigger("charger", action_add_queue_tail);
    } else {
        action_for_each_trigger("late-init", action_add_queue_tail);
    }

    /* run all property triggers based on current state of the properties */
    queue_builtin_action(queue_property_triggers_action, "queue_property_triggers");
    ...
```
调用***action_for_each_trigger()***函数用来指定的action加入到执行列表***action_queue***中，如果系统不在充电模式下，则把***late-init”action***添加到执行列表中；如果在充电模式，将charger加入到执行列表中。
***queue_builtin_action()***函数用来动态生成一个Action并插入到执行列表***action_queue***中。插入的Action由一个函数指针和一个表示名字的字符串组成。Android在以前的版本中直接调用这些函数来完成某些初始化的工作，但是这些函数可能汇集依赖init.rc里定义的一些命令和服务的执行情况。现在把这些初始化函数也是通过Action的形式插入到执行列表中，这样就能控制它们的执行顺序了。
插入的函数有：
`wait_for_coldboot_done_action()函数`：等待冷插拔设备初始化完成。
`mix_hwrng_into_linux_rng_action()函数`：从硬件RNG的设备文件/dev/hw_random中读取512字节并写到Linux RNG的设备文件/dev/urandom中。
`keychord_init_action()函数`：初始化组合键监听模块。
`console_init_action()函数`：在屏幕上显示Android字样的Logo。
`property_service_init_action()函数`：初始化属性服务，读取系统预置的属性值。
`signal_init_action()函数` : 初始化信号处理模块。
`check_startup_action()函数` : 检查是否已经完成Init进程的初始化，如果完成则删除.booting文件。
`queue_property_triggers_action()函数`: 检查Action列表中通过修改属性来触发的Action，查看相关的属性值是否已经设置，如果已经设置，则将该Action加入到执行列表中。
```c
    ...
    for(;;) {
        int nr, i, timeout = -1;
        execute_one_command();
        restart_processes();
        ...
    }
    return 0;
```
***main()函数***最后会进入到一个无限for循环，每次循环开始都会调用***execute_one_command()***函数来执行命令列表中的一条命令，同时调用***restart_process()***函数来启动服务进程。
```c
    ...
    if (!property_set_fd_init && get_property_set_fd() > 0) {
        ufds[fd_count].fd = get_property_set_fd();
        ufds[fd_count].events = POLLIN;
        ufds[fd_count].revents = 0;
        fd_count++;
        property_set_fd_init = 1;
    }
    if (!signal_fd_init && get_signal_fd() > 0) {
        ufds[fd_count].fd = get_signal_fd();
        ufds[fd_count].events = POLLIN;
        ufds[fd_count].revents = 0;
        fd_count++;
        signal_fd_init = 1;
    }
    if (!keychord_fd_init && get_keychord_fd() > 0) {
        ufds[fd_count].fd = get_keychord_fd();
        ufds[fd_count].events = POLLIN;
        ufds[fd_count].revents = 0;
        fd_count++;
        keychord_fd_init = 1;
    }
    ...
```
***Init***进程初始化系统后，会化身为守护进程来处理子进程的死亡信号，修改属性的请求和组合键盘事件。监听事件使用的是***poll**系统调用，使用poll前需要创建或获得用于监听这些事件的文件描述符，同时初始化***poll***函数调用需要的数据结构。
***poll()***调用可以设置等待超时的时间，参数为－1表示无限等待，参数为0表示要立刻返回，参数为正数表示要等待的时间，下面代码中计算的timeout时间就是用于poll调用的参数：
```c
...
if (process_needs_restart) {
    timeout = (process_needs_restart - gettime()) * 1000;
    if (timeout < 0)
        timeout = 0;
}
if (!action_queue_empty() || cur_action)
    timeout = 0;
#if BOOTCHART
    if (bootchart_count > 0) {
        if (timeout < 0 || timeout > BOOTCHART_POLLING_MS)
            timeout = BOOTCHART_POLLING_MS;
        if (bootchart_step() < 0 || --bootchart_count == 0) {
            bootchart_finish();
            bootchart_count = 0;
        }
    }
#endif
...    
```
timeout的初始值为－1。如果还有服务进程需要重启，会把timeout的值设置为下次启动服务的时间。如果命令列表中还有命令，则将timeout的时间设为0。要注意Init中并不是把命令队列中的所有命令一次就执行完毕，而是和poll在交替执行。这主要是考虑到执行所有命令时间太长，如果这期间有事件到来回你耽搁处理，因此，每执行一条列表中的命令就检查一次poll中的事件。
如果编译时定义了***BOOTACHART宏***，也需要定时唤醒进程，因此这里的timeout时间被设置为***BOOTACHART_POLLING_MS***。Bootchart是一个用可视化的方式对启动过程进行性能分析的工具。
```c
nr = poll(ufds, fd_count, timeout);
if (nr <= 0)
    continue;
for (i = 0; i < fd_count; i++) {
    if (ufds[i].revents & POLLIN) {
        if (ufds[i].fd == get_property_set_fd())
            handle_property_set_fd();
        else if (ufds[i].fd == get_keychord_fd())
            handle_keychord();
        else if (ufds[i].fd == get_signal_fd())
            handle_signal();
    }
}
```
调用***poll()***来监听事件的发生，如果监听到修改属性的事件，将会调用处理函数***handle_property_set_fd()***；如果监听到组合键盘消息，将会调用处理函数***handle_keychord()***；如果监听到信号，将会调用处理函数***handle_signal()***。
***poll()函数***和***select()函数***类似，用于监测多个等待事件，但是***poll()***比***select()***更加高效，如果没有事件，进程将会挂起，如果监听到任何一个事件发生，***poll()***将唤醒睡眠的进程。
