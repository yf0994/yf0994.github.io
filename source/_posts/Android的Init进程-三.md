---
title: Android的Init进程(三)
date: 2016-01-20 10:20:43
tags: Android系统
---
在***main()***函数的for循环中，会调用***restart_processes()***函数来启动服务列表中的服务进程。函数restart_processes()的代码如下所示：
```c
...
static void restart_processes(){
    process_needs_restart = 0;
    service_for_each_flags(SVC_RESTARTING,
                           restart_service_if_needed);
}
```
上面的代码中***service_for_each_flags()***函数会检查service_list列表中的每个服务，凡是带有SVC_RESTARTING标志的，都会使用该服务作为参数调用***restart_service_if_needed()***函数。***restart_serevice_if_needed()***函数中会调用***service_start()***函数来启动服务，我们主要关心的是init如何启动的服务进程，中间的代码就不分析了，直接看***service_start()***函数的代码。
1 重置Service结构中的标志
```c
void service_start(struct service *svc, const char *dynamic_args)
{
    ...
    /* starting a service removes it from the disabled or reset
     * state and immediately takes it out of the restarting
     * state if it was in there
     */
    svc->flags &= (~(SVC_DISABLED|SVC_RESTARTING|SVC_RESET|SVC_RESTART|SVC_DISABLED_START));
    svc->time_started = 0;

        /* running processes require no additional work -- if
         * they're in the process of exiting, we've ensured
         * that they will immediately restart on exit, unless
         * they are ONESHOT
         */
    if (svc->flags & SVC_RUNNING) {
        return;
    }
    ...
}
```
***SVC_DISABLED、SVC_RESTARTING、SVC_RESET、SVC_RESTART***这4个标志都是和启动进程相关，需要先清除掉。如果服务带有***SVC_RUNNING***标志，说明服务进程已经运行，这里就不重复启动了。
2 如果服务需要有控制台，但是还没有启动控制台就退出
```c
...
needs_console = (svc->flags & SVC_CONSOLE) ? 1 : 0;
if (needs_console && (!have_console)) {
    ERROR("service '%s' requires console\n", svc->name);
    svc->flags |= SVC_DISABLED;
    return;
}
...
```
上面的代码中***have_console***是一个全局变量，在函数***console_init_action()***中，如果打开设备文件***/dev/console***成功，它会被置为1。
3 检查service的二进制文件是否存在
```c
...
if (stat(svc->args[0], &s) != 0) {
    ERROR("cannot find '%s', disabling '%s'\n", svc->args[0], svc->name);
    svc->flags |= SVC_DISABLED;
    return;
}
...
```
4 检查SVC_ONESHOT参数
```c
...
if ((!(svc->flags & SVC_ONESHOT)) && dynamic_args) {
    ERROR("service '%s' must be one-shot to use dynamic args, disabling\n",
           svc->args[0]);
    svc->flags |= SVC_DISABLED;
    return;
}
...
```
5 设置安全上下文
```c
...
if (is_selinux_enabled() > 0) {
    if (svc->seclabel) {
        scon = strdup(svc->seclabel);
        if (!scon) {
            ERROR("Out of memory while starting '%s'\n", svc->name);
            return;
        }
    } else {
        char *mycon = NULL, *fcon = NULL;
        INFO("computing context for service '%s'\n", svc->args[0]);
        rc = getcon(&mycon);
        if (rc < 0) {
            ERROR("could not get context while starting '%s'\n", svc->name);
            return;
        }
        rc = getfilecon(svc->args[0], &fcon);
        if (rc < 0) {
            ERROR("could not get context while starting '%s'\n", svc->name);
            freecon(mycon);
            return;
        }
        rc = security_compute_create(mycon, fcon, string_to_security_class("process"), &scon);
        if (rc == 0 && !strcmp(scon, mycon)) {
            ERROR("Warning!  Service %s needs a SELinux domain defined; please fix!\n", svc->name);
        }
        freecon(mycon);
        freecon(fcon);
        if (rc < 0) {
            ERROR("could not get context while starting '%s'\n", svc->name);
            return;
        }
    }
}
...
```
6 fork子进程
```c
...
pid = fork();
...
```
7 准备环境变量
```c
...
if (properties_inited()) {
        get_property_workspace(&fd, &sz);
        sprintf(tmp, "%d,%d", dup(fd), sz);
        add_environment("ANDROID_PROPERTY_WORKSPACE", tmp);
    }
for (ei = svc->envvars; ei; ei = ei->next)
    add_environment(ei->name, ei->value);
...
```
在服务的选项中，如果有***setenv***选项，会将setenv的参数设置为服务进程的环境变量。注意这里把“属性”共享区域的文件句柄fd执行dup后的结果放到了***ANDROID_PROPERITY_WORKSPACE***环境变量中。这个fd在服务进程不能打开属性共享区的设备文件时使用。
8 创建socket
```c
...
for (si = svc->sockets; si; si = si->next) {
    int socket_type = (
            !strcmp(si->type, "stream") ? SOCK_STREAM :
                (!strcmp(si->type, "dgram") ? SOCK_DGRAM : SOCK_SEQPACKET));
    int s = create_socket(si->name, socket_type,
                          si->perm, si->uid, si->gid, si->socketcon ?: scon);
    if (s >= 0) {
        publish_socket(si->name, s);
    }
}

freecon(scon);
scon = NULL;
...
```
在服务选项中，如果有***socket***选项，这里会为服务创建参数中定义的socket，这里创建的socket是***PF_UNIX***类型的，这类socket的创建需要有***root***权限，因此要在执行***exec***调用之前创建出来。但是执行exec后服务进程不知道文件描述符也无法使用它。Android的解决办法是将描述符放到了一个环境变量中。环境变量的名字被定义为***ANDROID_SOCKET_XXX***，XXX是socket选项的名字。这样服务进程就能通过这个特殊名字的环境变量来得到socket的文件描述符了。为了让socket的文件描述符在执行exec后不被关闭，还需要对文件描述符执行***fcntl(fd, F_SETFD,0)***调用。这些都是在函数***publish_socket()***中完成的。
9 处理标准输入、标准输出、标准错误3个文件描述符
```c
...
if (needs_console) {
    setsid();
    open_console();
} else {
    zap_stdio();
}
...
```
如果需要使用控制台***console***。调用***open_console()***打开设备文件***/dev/console***，然后把标准输出、标准输入、标准错误重定向到该设备文件；否则调用***zap_stdio()***，把标准输出、标准输入、标准错误重定向到设备文件***/dev/null***。
10 执行exec
```c
...
if (!dynamic_args) {
        if (execve(svc->args[0], (char**) svc->args, (char**) ENV) < 0) {
            ERROR("cannot execve('%s'): %s\n", svc->args[0], strerror(errno));
        }
    } else {
        char *arg_ptrs[INIT_PARSER_MAXARGS+1];
        int arg_idx = svc->nargs;
        char *tmp = strdup(dynamic_args);
        char *next = tmp;
        char *bword;
            /* Copy the static arguments */
        memcpy(arg_ptrs, svc->args, (svc->nargs * sizeof(char *)));
        while((bword = strsep(&next, " "))) {
        arg_ptrs[arg_idx++] = bword;
        if (arg_idx == INIT_PARSER_MAXARGS)
            break;
    }
    arg_ptrs[arg_idx] = '\0';
    execve(svc->args[0], (char**) arg_ptrs, (char**) ENV);
}
...
```
执行完***exec***后，init进程的内存映像就被替换成新文件的映像了。上面的代码中调用的是***execve()***函数，它可以同时设置环境变量。