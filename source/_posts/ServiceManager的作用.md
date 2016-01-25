---
title: ServiceManager的作用
date: 2016-01-25 19:24:33
tags: Android系统
---
***ServiceManager***是***Binder***架构中用来解析***Binder***名称的模块。***ServiceManager***本身就是一个***Binder***服务。但是，***ServiceManager***并没有使用***libbinder***来构建***binder***服务，而是自己实现了一个简单的框架来直接和驱动通信。***ServiceManager***的源码位于目录***framework/native/cmds/servicemanager***下，主要由两个文件组成，一个是***binder.c***，用来实现简单的Binder通信功能，另一个文件是***service_manager.c***，用来实现***ServiceManager***的上层逻辑。
下面从ServiceManager的`main()`函数来开始分析，代码如下：
<!--more-->
```c
int main(int argc, char **argv){
    struct binder_state *bs;
    bs = binder_open(128*1024);
    if (!bs) {
        return -1;
    }
    if (binder_become_context_manager(bs)) {
        return -1;
    }
    selinux_enabled = is_selinux_enabled();
    sehandle = selinux_android_service_context_handle();
    if (selinux_enabled > 0) {
        if (sehandle == NULL) {
            abort();
        }
        if (getcon(&service_manager_context) != 0) {
            abort();
        }
    }
    union selinux_callback cb;
    cb.func_audit = audit_callback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);
    cb.func_log = selinux_log_callback;
    selinux_set_callback(SELINUX_CB_LOG, cb);
    svcmgr_handle = BINDER_SERVICE_MANAGER;
    binder_loop(bs, svcmgr_handler);
    return 0;
}
```
`main()`函数首先调用`binder_open()`打开`/dev/binder`设备和初始化系统，同时使用`mmap()`函数创建一块内存空间用于数据传输，参数***128*1024***就是这块空间的大小。接下来通过调用函数`binder_become_context_manager()`来把本进程设为binder框架的管理进程，`binder_become_context_manager()`函数的代码如下：
```c
int binder_become_context_manager(struct binder_state *bs){
    return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);
}
```
这个函数非常简单，直接通过`ioctl()`函数把控制命令***BINDER_SET_CONTEXT_MGR***发送到了驱动。
最后，`main()`函数调用`binder_loop()`函数来进入消息循环。调用`binder_loop()`会把函数指针***svcmgr_handler***作为参数传入。`binder_loop()`函数的主要工作就是从驱动读取命令，解析后再调用`svcmgr_handler()`来处理。函数`svcmgr_handler()`的代码如下:
```c
int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data *txn,
                   struct binder_io *msg,
                   struct binder_io *reply){
    struct svcinfo *si;
    uint16_t *s;
    size_t len;
    uint32_t handle;
    uint32_t strict_policy;
    int allow_isolated;
    //比较收到的handle是否等于svcmgr_handle
    if (txn->target.handle != svcmgr_handle)
        return -1;
    if (txn->code == PING_TRANSACTION)
        return 0;
	//检查收到的消息ID串
    strict_policy = bio_get_uint32(msg);
    s = bio_get_string16(msg, &len);
    if (s == NULL) {
        return -1;
    }
    if ((len != (sizeof(svcmgr_id) / 2)) ||
        memcmp(svcmgr_id, s, sizeof(svcmgr_id))) {
        fprintf(stderr,"invalid id %s\n", str8(s, len));
        return -1;
    }
    //检查SELinux权限
    if (sehandle && (selinux_status_updated() > 0)) {
        struct selabel_handle *tmp_sehandle = selinux_android_service_context_handle();
        if (tmp_sehandle) {
            selabel_close(sehandle);
            sehandle = tmp_sehandle;
        }
    }
    switch(txn->code) {
    case SVC_MGR_GET_SERVICE:
    //执行查询服务的功能
    case SVC_MGR_CHECK_SERVICE:
        s = bio_get_string16(msg, &len);
        if (s == NULL) {
            return -1;
        }
        handle = do_find_service(bs, s, len, txn->sender_euid, txn->sender_pid);
        if (!handle)
            break;
        bio_put_ref(reply, handle);
        return 0;
	 //执行注册服务的功能
    case SVC_MGR_ADD_SERVICE:
        s = bio_get_string16(msg, &len);
        if (s == NULL) {
            return -1;
        }
        handle = bio_get_ref(msg);
        allow_isolated = bio_get_uint32(msg) ? 1 : 0;
        if (do_add_service(bs, s, len, handle, txn->sender_euid,
            allow_isolated, txn->sender_pid))
            return -1;
        break;
	 //执行获取服务列表的功能
    case SVC_MGR_LIST_SERVICES: {
        uint32_t n = bio_get_uint32(msg);
        if (!svc_can_list(txn->sender_pid)) {
            ALOGE("list_service() uid=%d - PERMISSION DENIED\n",
                    txn->sender_euid);
            return -1;
        }
        si = svclist;
        while ((n-- > 0) && si)
            si = si->next;
        if (si) {
            bio_put_string16(reply, si->name);
            return 0;
        }
        return -1;
    }
    default:
        ALOGE("unknown code %d\n", txn->code);
        return -1;
    }
    //发送返回消息
    bio_put_uint32(reply, 0);
    return 0;
}
```
从函数`svcmgr_handle()`的代码来看，***ServiceManager***提供3中服务功能：***注册Binder服务，查询Binder服务和获取Binder服务列表***。
注册binder服务的处理函数`do_add_service()`的代码如下：
```c
int do_add_service(struct binder_state *bs,
                   const uint16_t *s, size_t len,
                   uint32_t handle, uid_t uid, int allow_isolated,
                   pid_t spid){
    struct svcinfo *si;
    if (!handle || (len == 0) || (len > 127))
        return -1;
	//检查调用进程是否有权限进行注册
    if (!svc_can_register(s, len, spid)) {
        ALOGE("add_service('%s',%x) uid=%d - PERMISSION DENIED\n",
             str8(s, len), handle, uid);
        return -1;
    }
    //查看要注册的服务名是否已经存在
    si = find_svc(s, len);
    if (si) {
	//如果存在，先把以前的Binder对象的引用计数减一
        if (si->handle) {
            svcinfo_death(bs, si);
        }
		//把列表节点中的ptr换成新的值
        si->handle = handle;
    } else {
	    //如果服务不存在，则生成新的列表项，初始化后加入列表
        si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));
        if (!si) {
            return -1;
        }
        si->handle = handle;
        si->len = len;
        memcpy(si->name, s, (len + 1) * sizeof(uint16_t));
        si->name[len] = '\0';
        si->death.func = (void*) svcinfo_death;
        si->death.ptr = si;
        si->allow_isolated = allow_isolated;
        si->next = svclist;
        svclist = si;
    }
    //增加Binder服务的引用计数
    binder_acquire(bs, handle);
    //告诉驱动接收服务的死亡通知
    binder_link_to_death(bs, handle, &si->death);
    return 0;
}
```
`do_add_services()`函数的流程是首先检查调用进程是否有权限注册，接着查看需要注册的服务是否已经存在，如果存在就把原来的Binder服务在驱动中的引用计数减一，不存在则新创建一个***svcinfo***的结构，把服务的名称等信息填入结构中，然后把结构加入到服务列表***svclist***中，最后通知内核把***Binder服务***的引用计数加一，并且需要接受该Binder服务的死亡通知。
	检查调用进程权限的函数`svc_can_register()`，它的代码如下：
```c
static int svc_can_register(const uint16_t *name, size_t name_len, pid_t spid){
    const char *perm = "add";
    return check_mac_perms_from_lookup(spid, perm, str8(name, name_len)) ? 1 : 0;
}
```
`svc_can_register()`函数通过调用函数`check_mac_perms_from_lookup()`检查进程权限的规则发生了变化，其代码如下：
```c
static bool check_mac_perms_from_lookup(pid_t spid, const char *perm, const char *name){
    bool allowed;
    char *tctx = NULL;
    if (selinux_enabled <= 0) {
        return true;
    }
    if (!sehandle) {
        abort();
    }
    if (selabel_lookup(sehandle, &tctx, name, 0) != 0) {
        return false;
    }
    allowed = check_mac_perms(spid, tctx, perm, name);
    freecon(tctx);
    return allowed;
}
```
`check_mac_perms_from_lookup()`函数对`SElinux`的环境和进程名称name检查后，又调用了`check_mac_perms()`函数来检查权限，函数代码如下：
```c
static bool check_mac_perms(pid_t spid, const char *tctx, const char *perm, const char *name){
    char *sctx = NULL;
    const char *class = "service_manager";
    bool allowed;
    if (getpidcon(spid, &sctx) < 0) {
        return false;
    }
    int result = selinux_check_access(sctx, tctx, class, perm, (void *) name);
    allowed = (result == 0);
    freecon(sctx);
    return allowed;
}
```
`check_mac_perms()`函数首先调用`getpidcon()`函数得到进程的安全上下文，然后调用`selinux_check_access()`函数来判断进程是否有权限进行参数perm中定义的操作。
	执行查询服务的函数是`do_find_service()`，代码如下:
```c
uint32_t do_find_service(struct binder_state *bs, const uint16_t *s, size_t len, uid_t uid, pid_t spid){
    struct svcinfo *si;
    if (!svc_can_find(s, len, spid)) {
        return 0;
    }
    si = find_svc(s, len);
    if (si && si->handle) {
        if (!si->allow_isolated) {
        	//如果Binder服务不允许服务从沙箱中访问，执行下面的检查
            uid_t appid = uid % AID_USER;
            if (appid >= AID_ISOLATED_START && appid <= AID_ISOLATED_END) {
                return 0;
            }
        }
        return si->handle;
    } else {
        return 0;
    }
}
```
`do_find_service()`函数的主要工作是搜索列表，返回查找到的服务。注意，这里面有一段判断***uid***的代码。在调用***ServiceManager***的`addBinder()`服务时有个参数时***allow_isolated***，用来指定服务是否允许从隔离的进程中访问。这里的代码应该是在判断调用进程是否是一个隔离进程。方法是先用***uid***对***AID_USER(100000)***取模，再比较结果是否在***AID_ISOLATED_START(99000)***和***AID_ISOLATED_END(99999)***之间，普通的用户进程号正是位于这个区间，这表明某个服务可以通过***allowed_isolated***属性来控制是否允许一般的用户进程来使用这个服务。