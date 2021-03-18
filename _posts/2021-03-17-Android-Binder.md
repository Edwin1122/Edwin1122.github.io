---
title: Anroid Binder机制
author: Edwin
date: 2021-03-17 10:50:43  +0800
categories: [Android]
tags: [Binder]
---

## 一. 前言

Android 的 Binder 机制, 是作为 Android 进阶学习中一个必学的知识点。 之前, 通过学习 《Android 开发艺术探索》以及 网上的一些博客, 大致了解 Android Binder 机制的过程, 自以为已掌握 Binder 机制的大致过程, 写下博客 Android—-IPC机制（Binder） , 但是实际上只是了解了皮毛，没有深入整个流程，于是重新学习，将新的体会记录成这篇 进阶篇。

阅读这篇文章之前需要具备基本的 Binder 机制的流程, 具体可通过这些博客学习

[Android—-IPC机制（Binder）](https://blog.csdn.net/m0_38089373/article/details/81206893)

[Binder学习指南](http://weishu.me/2016/01/12/binder-index-for-newer/)

[图文详解 Android Binder跨进程通信的原理](https://www.jianshu.com/p/4ee3fd07da14)

手动实现 Binder 的通信, 不依靠 AIDL 自动生成。

## 二. 跨进程

### (一). 用户空间/内核空间

内核空间是 Linux 系统中内核的运行空间, 在内核空间可以随意访问系统的资源, 系统内核对资源进行同一的管理。
用户空间是每个用户进程运行的空间, 这个空间是进程独有的, 不能和其他进程进行内存共享, 主要是用于保存进程中的数据等, 由于可能存在有恶意的用户进程对系统进行破坏, 且系统资源是宝贵的, 这就需要保证系统的安全, 所以一个进程的用户空间不能直接访问系统资源。
那么一个用户空间如何访问系统资源呢？

通过内核提供的接口, 向内核发出指令, 然后交由内核去访问系统资源。

大致情况如下图：

![alt](https://yishengma.github.io/2019/01/24/Android-Binder%E6%9C%BA%E5%88%B6%EF%BC%88%E8%BF%9B%E9%98%B6%E7%AF%87%EF%BC%89/1240)

### (二). Binder 驱动

虽然两个进程的用户空间互不共享, 但是一个进程除了有用户空间, 还有内核空间, 且内核空间是共有的, 所以两个进程想要进行通信, 那么就可以通过一个在内核空间的中介进行数据的交互，这个中介就是 Binder 驱动。

Binder 驱动是 Android 在 Linux 内核上添加的一个内核模块，作为内核的一部分在内核空间上运行里运行, 两个进程通过这个在内核空间运行的 Binder 驱动就可以进行通信。

![alt](https://yishengma.github.io/2019/01/24/Android-Binder%E6%9C%BA%E5%88%B6%EF%BC%88%E8%BF%9B%E9%98%B6%E7%AF%87%EF%BC%89/12401)

### (三). 内核启动

Binder 驱动是运行在内核空间的, 在内核启动的时候 就会将 Binder 注册成 misc device 类型的驱动, 也就是内核驱动。

``` java

static int __init binder_init(void)
{
    int ret;
    char *device_name, *device_names;
    struct binder_device *device;
    struct hlist_node *tmp;
    
    while ((device_name = strsep(&device_names, ","))) {
        //初始化 Binder 设备
        ret = init_binder_device(device_name);
        if (ret)
            goto err_init_binder_device_failed;
    }

    return ret;
static int __init init_binder_device(const char *name)
{
    int ret;
    struct binder_device *binder_device;
    
    //1.为 Binder 设备开辟空间
    binder_device = kzalloc(sizeof(*binder_device), GFP_KERNEL);
    if (!binder_device)
        return -ENOMEM;
    //2.初始化 Binder 
    binder_device->miscdev.fops = &binder_fops;
    binder_device->miscdev.minor = MISC_DYNAMIC_MINOR;
    binder_device->miscdev.name = name;

    binder_device->context.binder_context_mgr_uid = INVALID_UID;
    binder_device->context.name = name;
    //2.注册 Binder  设备
    ret = misc_register(&binder_device->miscdev);
    if (ret < 0) {
        kfree(binder_device);
        return ret;
    }

    hlist_add_head(&binder_device->hlist, &binder_devices);

    return ret;
}

```

![alt](https://yishengma.github.io/2019/01/24/Android-Binder%E6%9C%BA%E5%88%B6%EF%BC%88%E8%BF%9B%E9%98%B6%E7%AF%87%EF%BC%89/12402)

## 三. ServiceManager

有大致了解过的 Binder 的人都知道 Binder 机制涉及到四个对象

客户端
服务端
Binder 驱动
ServiceManager
前三个相信大家都清楚是什么, 至于第四个 ServiceManager, 从名字上就可以看出, 它的作用就是对服务端进行管理, 进程间的通信时一个常见的功能, 每个进程都可以作为客户端或服务端通过 Binder 驱动参与进程间的通信, 而且通信的前提就是找到通信的另一方, 并建立联系, ServiceManager 的作用就在于此，其记录了所有的服务端, 当一个客户端想要和服务端进行通信的时候，首先就会去 ServiceManager 找到对应的服务端。这好比打电话, Binder 驱动就是信号基站, 而 ServiceManager 就是一个通信录，进行 A 要和 进程 B 进行通信, 首先就要去 通信录找到进程 B 的电话号码是多少, 然后就可通过这个电话号码, 将数据通过信号基站传递给进程 B ，这就完成了进程通信。

但是 ServiceManager 本身也是一个进程, 这是一个 native 进程, 客户端获取电话号码的过程也本应该是一个跨进程的通信(实际上并没有), 那么客户端是如何首先获取 ServiceManager 的对应的 Binder 引用的呢?

这就是本篇文章要解决的第一个问题。

### (一). ServiceManager 的启动

ServiceManager 是一个 native 进程, 其源码自然是 C/C++ , 但是不用担心, 这里只关注一些重要的地方, 通过函数的名字就可以看出其作用。

1.init 进程
在 Android 系统中 init 进程是系统启动的第一个进程, 在内核启动后就会启动 init 进程 , 它启动后就会去解析 init.c 这个文件，然后启动其他的 native 进程, 而 ServiceManager 作为一个 native 进程, 自然也是由它启动的。

在解析 init.rc 文件的时候去启动 servicemanager.rc

``` 

service servicemanager /system/bin/servicemanager
    class core animation
    user system
    group system readproc
    critical
    onrestart restart healthd
    onrestart restart zygote
    onrestart restart audioserver
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart inputflinger
    onrestart restart drm
    onrestart restart cameraserver
    onrestart restart keystore
    onrestart restart gatekeeperd
    writepid /dev/cpuset/system-background/tasks
    shutdown critical

```

2.servicemanager 的启动
servicemanager 的启动 的 启动从 servicemanager.c 文件中的 main 方法开始的

``` c

// native\native\cmds\servicemanager\servicemanager.c

int main(int argc, char** argv)
{
    struct binder_state *bs;
    
    // 1.打开 Binder 驱动
    //并设置 servicemanager 进行地址映射的大小为  128 KB 
    bs = binder_open(driver, 128*1024);
    
    // 2.将 servicemanager 驱动设置为唯一的管理者，也就是系统内核只有一个servicemanager
    if (binder_become_context_manager(bs)) {
        ALOGE("cannot become context manager (%s)\n", strerror(errno));
        return -1;
    }
    //3.启动循环，不断处理客户端的请求
    binder_loop(bs, svcmgr_handler);

    return 0;
}

```

(1).binder_open

``` c
 // native\native\cmds\servicemanager\binder.c
 
struct binder_state *binder_open(const char* driver, size_t mapsize)
{
    struct binder_state *bs;
    struct binder_version vers;

    bs = malloc(sizeof(*bs));
   
    //进入系统内核 打开 Binder 设备
    bs->fd = open(driver, O_RDWR | O_CLOEXEC);
   
    //地址映射
    bs->mapsize = mapsize;
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
    
    return bs;

```

系统内核打开 Binder 设备

``` c
static int binder_open(struct inode *nodp, struct file *filp)
{
    struct binder_proc *proc;
    struct binder_device *binder_dev;

    //为当前进程 创建一个 proc 结构体
    //这里就是为 ServiceManager 创建一个结构体
    proc = kzalloc(sizeof(*proc), GFP_KERNEL);
    //初始化结构体的 todo 队列
    
    INIT_LIST_HEAD(&proc->todo);  
    //初始化结构体的 wait 队列
    init_waitqueue_head(&proc->wait);
    
    //加锁
    binder_lock(__func__);

    binder_stats_created(BINDER_STAT_PROC);
    //将当前进程,也就是  ServiceManager 的结构体添加到 Binder 的一个全局队列中
    hlist_add_head(&proc->proc_node, &binder_procs);
    proc->pid = current->group_leader->pid;
    INIT_LIST_HEAD(&proc->delivered_death);
    filp->private_data = proc;

   //释放锁
    binder_unlock(__func__);

    return 0;
}

```

首先我们需要知道的是每一个作为服务端的进程, 都对应在 Binder 驱动中的一个节点
todo 队列保存的是请求的事务，如果一个客户端发起一个请求, 那么就会向 todo 队列添加一个事务。
wait 队列, 将客户端发起请求后等待响应添加到 wait 等待队列, 待服务端执行完后就唤醒。
由于发起 Binder 线程的情况有很多但是每个进程只创建一个对应的结构体添加到链表里，因此就需要加锁。

ServiceManager 注册后的情况就是如下图：

![alt](https://yishengma.github.io/2019/01/24/Android-Binder%E6%9C%BA%E5%88%B6%EF%BC%88%E8%BF%9B%E9%98%B6%E7%AF%87%EF%BC%89/12403)

(2)将 servicemanager 驱动设置为唯一的管理者, 并设置为 0 号 handler
前面说过 servicemanager 管理者其他服务端, 客户端可以通过 serviceManger 查询对应的 Binder 服务端, 这个查询的过程就是寻找服务端进程对应的在 binder 中的 “号码”, 这些号码由 servicemanager 统一管理, 那么 servicemanager 自己的号码注册为 0 号。

这好比网络传输中获取服务端的 ip 地址首先要通过 DNS 查询, 而这个 DNS 的 ip 地址都是已知的。

这里的就是 0 号, “号码”称为 handler

``` c

// 
   if (binder_become_context_manager(bs)) {

        ALOGE("cannot become context manager (%s)\n", strerror(errno));
        return -1;
    }

int binder_become_context_manager(struct binder_state *bs)
{

    return ioctl(bs->fd, BINDER_SET_CONTEXT_MGR, 0);

}
//调用到内核的 ioctl, 参数是 BINDER_SET_CONTEXT_MGR

static int binder_ioctl_set_ctx_mgr(struct file *filp)
{

    int ret = 0;
    struct binder_proc *proc = filp->private_data;
    struct binder_context *context = proc->context;

    //新建一个节点 ，第二个参数是 0 
    context->binder_context_mgr_node = binder_new_node(proc, 0, 0); 
    
    return ret;

}
static struct binder_node *binder_new_node(struct binder_proc *proc, 

    binder_uintptr_t ptr,
    binder_uintptr_t cookie)

{

    struct rb_node **p = &proc->nodes.rb_node;
    struct rb_node *parent = NULL;
    struct binder_node *node;

    //插入这个节点
    while (*p) {
        parent = *p;
        node = rb_entry(parent, struct binder_node, rb_node);

        if (ptr < node->ptr)
            p = &(*p)->rb_left;
        else if (ptr > node->ptr)
            p = &(*p)->rb_right;
        else
            return NULL;
    }

    node->proc = proc;
    
    //ptr 指的就是用户空间的一个引用，因此这个参数为 0
    //所以 serviceManger 在客户端的对应的 binder 对象就是 0 号引用获得的。
    node->ptr = ptr;
    
    return node;

}

```

### (二). ServiceManager Binder 的获取

我们知道 ServiceManager 作为一个进程, 客户端于其的通信也是一种进程通信, 从上面可以知道 ServiceManager 对应的 Binder 是 0 号引用, 那么客户端是是如何获取的到 0 号引用即 ServiceManager 的 Binder 对象呢？

以 bindService 为例

``` java

//base\core\java\android\app\ContextImpl.java
  private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags, Handler handler, UserHandle user) {
           ...
           //bindService 是通过 ActivityManager.getService() 去调用的
            int res = ActivityManager.getService().bindService(
                mMainThread.getApplicationThread(), getActivityToken(), service,
                service.resolveTypeIfNeeded(getContentResolver()),
                sd, flags, getOpPackageName(), user.getIdentifier());
           ...
    }

```

在 ActivityManager 中

``` java

//base\core\java\android\app\ActivityManager.java
public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

    private static final Singleton<IActivityManager> IActivityManagerSingleton =
        new Singleton<IActivityManager>() {
            @Override
            protected IActivityManager create() {
            //要获取 IActivityManager 对象,就要通过 ServiceManager 去查询并获取
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                final IActivityManager am = IActivityManager.Stub.asInterface(b);
                return am;
            }
        };
// base\core\java\android\os\ServiceManager.java

//通过一个 服务的名字去获取对应的 Binder 
  public static IBinder getService(String name) {
      try {
          IBinder service = sCache.get(name);
          if (service != null) {
              return service;
          } else {
              //这里就是先通过 getIServiceManager 去查询,再返回
              return
              Binder.allowBlocking(getIServiceManager().getService(name));
          }
      } catch (RemoteException e) {
          Log.e(TAG, "error in getService", e);
      }
      return null;
  }
  
  
  //获取 ServiceManager Binder 对象 IServiceManager 就在这个 方法里
 private static IServiceManager getIServiceManager() {
      if (sServiceManager != null) {
          return sServiceManager;
      }

      // Find the service manager
      //关键是这里的 BinderInternal.getContextObject();
      sServiceManager = ServiceManagerNative
              .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
      return sServiceManager;
  }
//显然对应的方法就在 BinderInternal.getContextObject();

//base\core\java\com\android\internal\os\BinderInternal.java
 public static final native IBinder getContextObject();

 ```

这是个 native 方法对应 c++ 实现如下

``` c++
//base\core\jni\android_util_Binder.cpp
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{

    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    return javaObjectForIBinder(env, b);

}

``` 

android_util_Binder 的作用就是将 c/c++ 的 Binder 对象转换为 java 层。
接着看 ProcessState 的 getContextObject 方法

``` c++

// native\native\libs\binder\ProcessState.cpp
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    return getStrongProxyForHandle(0);
}

//这个 0 就是之前说过的 ServiceManager 在 Binder 驱动中对应的引用号 hanler，这里就为 0 .即 ServiceManager 为 0 号。

sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;
    
    handle_entry* e = lookupHandleLocked(handle);

    if (e != nullptr) {
        
        IBinder* b = e->binder;
       if (b == nullptr || !e->refs->attemptIncWeak(this)) {
       
       //当 handle == 0 的时候
            if (handle == 0) {
            //在 handle == 0 的情况下
            //因为 ServiceManager 唯一我们在创建其代理类的时候
            //不需要持有其引用的对象，因为其引用号已知为 0 
            //所以通过简单的远程访问确定其已经注册否则 
            //ServiceManager 就不能使用。
                Parcel data;
                status_t status = IPCThreadState::self()->transact(0, IBinder::PING_TRANSACTION, data, nullptr, 0);
                if (status == DEAD_OBJECT)
                   return nullptr;
            }
    }

    return result;
}

```

在说明 new BpBinder(handle) 的作用之前, 这里记录下一个疑问

ServiceManager 对应在 native 层的是 IServiceManager.cpp 但是从上面的调用链来看并没有涉及到, 可能是 c/c++ 的语法或者是有部分源码看漏了, 这里先借用其他文章的思路, 直接看 IServiceManager。

``` c++
//native\native\libs\binder\IServiceManager.cpp
sp<IServiceManager> defaultServiceManager()
{

    if (gDefaultServiceManager != nullptr) return gDefaultServiceManager;
    {
        AutoMutex _l(gDefaultServiceManagerLock);
        while (gDefaultServiceManager == nullptr) {
            gDefaultServiceManager = interface_cast<IServiceManager>(
                ProcessState::self()->getContextObject(nullptr));
            if (gDefaultServiceManager == nullptr)
                sleep(1);
        }
    }
    return gDefaultServiceManager;

}

``` 

这个获取 的过程可以参考 浅谈Android系统进程间通信（IPC）机制Binder中的Server和Client获得Service Manager接口之路

总结

一般跨进程的通信首先要需要通过 ServiceManager 获取对方的代理接口才能通信,但是 对于 ServiceManager 自己的代理接口的获取过程不需要跨进程,因为其引用号已知为 0.

如果是同一个进程的的通信,客户端 透过 Binder 驱动向 服务端 请求一个 Binder 代理对象时，Binder 驱动发现它们是同一个进程，就向 客户端 进程返回一个 Binder 本地对象，而不是 Binder 代理对象。

![alt](https://yishengma.github.io/2019/01/24/Android-Binder%E6%9C%BA%E5%88%B6%EF%BC%88%E8%BF%9B%E9%98%B6%E7%AF%87%EF%BC%89/12404)

## 四.通信

在客户端拿到 IServiceManager 的 Binder 对象后就将其封装成为一个 代理对象 Proxy

``` java

//base\core\java\android\os\ServiceManager.java
private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }
        // Find the service manager
        sServiceManager = ServiceManagerNative
                .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
        return sServiceManager;
    }

```

下面具体看 ServiceManagerNative的asInterface

``` java
static public IServiceManager asInterface(IBinder obj)
    {
        if (obj == null) {
            return null;
        }
        IServiceManager in =
            (IServiceManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }
        
        return new ServiceManagerProxy(obj);
    }

```

这部分代码对于手动写过 Binder 客户端服务端的人都比较熟悉, 主要是判断获取的接口是位于同一进程还是 不同进程，如果是统一进程, 则不需要进行跨进程的通信, 所以返回本地接口/对象就像就行，如果是不同进程, 就返回一个代理对象, 下面看如何通过跨进程通信。

以向 SericeManager 获取服务为例

``` java

class ServiceManagerProxy implements IServiceManager {
    public ServiceManagerProxy(IBinder remote) {
        mRemote = remote;
    }
    
    public IBinder asBinder() {
        return mRemote;
    }
    
    public IBinder getService(String name) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IServiceManager.descriptor);
        data.writeString(name);
        mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0);
        IBinder binder = reply.readStrongBinder();
        reply.recycle();
        data.recycle();
        return binder;
    }

```

下面就对 getService 的几个变量做一下说明

data , 向 SericeManager (服务端)传递的参数 上面 writeString 写入数据，writeInterfaceToken 写入接口标识

reply , SericeManager (服务端 ) 执行后的往 reply 写入返回值
mRemote.transact 进行跨进程传输，这个返回会阻塞于 Binder 线程池，因为要等待其 SericeManager (服务端) 返回

reply.readStrongBinder() 服务端返回后唤醒当前 Binder 线程, 读取查询后的 Binder 对象。

1. Parcel

    Parcel 是传递的数据的载体, 在 java 层和 native 都有对应的类。主要的传输过程都在 transact 函数中。

2. transact

``` c++

//base\core\jni\android_util_Binder.cpp
static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj, 

        jint code, jobject dataObj, jobject replyObj, jint flags) // throws RemoteException

{

    //将 java 层的对应的 Parcel 对应到 native 层
    Parcel* data = parcelForJavaObject(env, dataObj);
    Parcel* reply = parcelForJavaObject(env, replyObj);

     //获取native 层的Binder 对象
    IBinder* target = getBPNativeData(env, obj)->mObject.get();
    //调用 native 层的 Binder 的 方法
    status_t err = target->transact(code, *data, reply, flags);
    
    reply->print();
    return JNI_FALSE;

``` 

最后都会调用 Binder 驱动的 binder_transaction 方法。

``` c++

static void binder_transaction(struct binder_proc *proc,
               struct binder_thread *thread,
               struct binder_transaction_data *tr, int reply){
   
            // 由 handle 找到相应 binder 引用, 由binder引用 找到相应 binder 节点
            ref = binder_get_ref(proc, tr->target.handle);
            target_node = ref->node;
        } else {
            target_node = binder_context_mgr_node;
        }
        // 由 binder 节点 找到相应 binder proc 结构体
        target_proc = target_node->proc;
    }

    //从目标进程 proc 中分配内存空间
    t->buffer = binder_alloc_buf(target_proc, tr->data_size,
        tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));

    //分别拷贝用户空间的 binder_transaction_data  到目标进程的 binder 缓冲区
    copy_from_user(t->buffer->data,
        (const void __user *)(uintptr_t)tr->data.ptr.buffer, tr->data_size);
    copy_from_user(offp,
        (const void __user *)(uintptr_t)tr->data.ptr.offsets, tr->offsets_size);

   ...
    return;
}

```

![alt](https://yishengma.github.io/2019/01/24/Android-Binder%E6%9C%BA%E5%88%B6%EF%BC%88%E8%BF%9B%E9%98%B6%E7%AF%87%EF%BC%89/12405)

通过这种方式数据的传递就从 A 进程到了 Service 进程。

关于 Binder 的机制, 这里只是记录笔者的自己理一遍的思路，如果想要深入学习，可以参考一下博客，本篇也是参考其博客才写下的。

[Gityuan 的 Binder系列](http://gityuan.com/tags/#binder)

[Binder学习指南](http://weishu.me/2016/01/12/binder-index-for-newer/)

[Android Bander设计与实现 - 设计篇](https://blog.csdn.net/universus/article/details/6211589#t1)

[一篇文章了解相见恨晚的 Android Binder 进程间通讯机制](https://blog.csdn.net/luoshengyang/article/details/6618363)

[Android进程间通信（IPC）机制Binder简要介绍和学习计划](https://blog.csdn.net/luoshengyang/article/details/6618363)

[图文详解 Android Binder跨进程通信的原理](https://www.jianshu.com/p/4ee3fd07da14)

摘自 [Anrodid Binder机制](https://yishengma.github.io/2019/01/24/Android-Binder%E6%9C%BA%E5%88%B6%EF%BC%88%E8%BF%9B%E9%98%B6%E7%AF%87%EF%BC%89/)
