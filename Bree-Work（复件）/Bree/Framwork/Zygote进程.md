# 一、简介

Zygote是Android中最重要的一个进程，Zygote进程和Init进程、SystemServer进程是Android最重要的三大进程。
Zygote是Android系统创建新进程的核心进程，负责启动Dalvik虚拟机，加载一些必要的系统资源和系统类，启动system_server进程，随后进入等待处理app应用请求。
在Android系统中，**应用程序进程都是由Zygote进程孵化出来的，而Zygote进程是由Init进程启动的**。Zygote进程在启动时会创建一个Dalvik虚拟机实例，每当它孵化一个新的应用程序进程时，都会将这个Dalvik虚拟机实例复制到新的应用程序进程里面去，从而使得每一个应用程序进程都有一个独立的Dalvik虚拟机实例。


Zygote涉及的类
```
frameworks/base/cmds/app_process/app_main.cpp
frameworks/base/core/jni/AndroidRuntime.cpp
frameworks/base/core/java/com/android/internal/os/
  - Zygote.java
  - ZygoteInit.java
  - ZygoteServer.java
  - ZygoteConnection.java

```


# 二、Zygote启动


![[Zygote启动流程图.png]]

# 三、总结

所有的进程都由Zygote创建，zygote主要用来孵化system_server进程和应用程序进程。在孵化出第一个进程system_server后通过runSelectLoop等待并处理消息，分裂应用程序进程仍由system_server控制，等待 AMS 给他发消息（告诉 zygote 创建进程），如app启动时创建子进程。

从AndroidRuntime到ZygoteInit，主要分为3大过程：

```
1、创建虚拟机——startVm():调用JNI虚拟机创建函数
2、注册JNI函数——startReg()：前面已经创建虚拟机，这里给这个虚拟机注册一些JNI函数（后续java世界用到的函数是native实现，这里需要提前注册注册这些函数）
3、此时就要执行CallStaticViodMethod，通过这个函数将进入android精心打造的java世界，这个函数将调用com.android.internal.os.ZygoteInit的main函数
```


在 ZygoteInit.main函数中进入Java世界，主要有4个关键步骤：

```
1、预加载类和资源——preload()
主要是preloadClasses和preloadResources，其中preloadClasses一般是加载时间超过1250ms的类，因而需要在zygote预加载
2、建立IPC通信服务——初始化ZygoteServer，内部初始化了ZygoteSocket
zygote及系统中其他程序的通信并没有使用Binder，而是采用基于AF_UNIX类型的Socket，作用正是建立这个Socket
3、启动system_server——forkSystemServer()
这个函数会创建Java世界中系统Service所驻留的进程system_server,该进程是framework的核心，也是zygote孵化出的第一个进程。如果它死了，就会导致zygote自杀。
4、等待请求——runSelectLoop()
zygote从startSystemServer返回后，将进入第四个关键函数runSelectLoop，在第一个函数ZygoteServer中注册了一个用于IPC的Socket将在这里使用，这里Zygote采用高效的I/O多路复用机制，保证在没有客户端请求时或者数据处理时休眠，否则响应客户端的请求。等待 AMS 给他发消息（告诉 zygote 创建进程）。此时zygote完成了java世界的初创工作，调用runSelectLoop便开始休眠了，当收到请求或者数据处理便会随时醒来，继续工作。
```



# 四、Zygote相关问答

## 1.init进程作用是什么

init进程起着承上启下的作用，Android本身是基于Linux而来的，init进程是Linux系统中用户空间的第一个进程。init进程属于一个守护进程，准确的说，它是Linux系统中用户控制的第一个进程，它的进程号为1（进程号为0的为内核进程），它的生命周期贯穿整个Linux内核运行的始终。Android中所有其它的进程共同的鼻祖均为init进程。
Android Q(10.0) 的init入口函数由原先的init.cpp 调整到了main.cpp，把各个阶段的操作分离开来，使代码更加简洁命令。
作为天子第1号进程，init被赋予了很多重要的职责，主要分为三个阶段：

```
init进程第一阶段做的主要工作是挂载分区，创建设备节点和一些关键目录，初始化日志输出系统,启用SELinux安全策略。
init进程第二阶段主要工作是初始化属性系统，解析SELinux的匹配规则，处理子进程终止信号，启动系统属性服务，可以说每一项都很关键，如果说第一阶段是为属性系统，SELinux做准备，那么第二阶段就是真正去把这些功能落实。
init进行第三阶段主要是解析init.rc来启动其他进程，进入无限循环，进行子进程实时监控(守护)。

```

其中第三阶段通过initrc启动其他进程，我们常见的比如启动Zygote进程、启动SeviceManager进程等。

## 2.Zygote进程最原始的进程是什么进程(或者Zygote进程由来)
Zygote最开始是app_process，它是在 init 进程启动时被启动的，在app_main.cpp才被修改为 Zygote。

## 3.Zygote 是在内核空间还是在用户空间？
因为 init 进程的创建在用户空间，而 Zygote 是由 init 进程创建启动的，所以Zygote是在用户空间。

## 4.Zygote为什么需要用到Socket通信而不是Binder
Zygote是Android中的一个重要进程，它是启动应用程序进程的父进程。Zygote使用Socket来与应用程序进程进行通信，而不是使用Android中的IPC机制Binder，这是因为Socket和Binder有不同的优缺点，而在Zygote进程中使用Socket可以更好地满足Zygote进程的需求。

```
Zygote 用 binder 通信会导致死锁
假设 Zygote 使用 Binder 通信，因为 Binder 是支持多线程的，存在并发问题，而并发问题的解决方案就是加锁，如果进程 fork 是在多线程情况下运行，Binder 等待锁在锁机制下就可能会出现死锁。

Zygote 用 binder 通信会导致读写错误
根本原因在于要 new 一个 ProcessState 用于 Binder 通信时，需要 mmap 申请一片内存用以提供给内核进行数据交换使用。而如果直接 fork 了的话，子进程在进行 binder 通信时，内核还是会继续使用父进程申请的地址写数据，而此时会触发子进程 COW（Copy on Write），从而导致地址空间已经重新映射，而子进程还尝试访问之前父进程 mmap 的地址，会导致 SIGSEGV、SEGV_MAPERR段错误。

Zygote初始化时，Binder还没开始初始化。
Socket具有良好的跨平台性，能够在不同的操作系统和语言之间进行通信。这对于Zygote进程来说非常重要，因为它需要在不同的设备和架构上运行，并且需要与不同的应用程序进程进行通信。使用Socket可以让Zygote进程更加灵活和可扩展，因为它不需要考虑Binder所带来的特定限制和要求。

Socket具有简单的API和易于使用的特点。Zygote进程需要快速启动并与应用程序进程建立通信，Socket提供了快速、可靠的通信方式，并且使用Socket API也很容易实现。相比之下，Binder需要更多的配置和维护工作，这对于Zygote进程来说可能会增加不必要的复杂性和开销。

Socket在数据传输时具有更低的延迟和更高的吞吐量，这对于Zygote进程来说非常重要。Zygote进程需要在较短的时间内启动应用程序进程，并且需要传输大量的数据和代码，Socket的高性能和低延迟使其成为更好的选择。

```

总之，Zygote进程使用Socket而不是Binder是基于其优点和需求而做出的选择。虽然Binder在Android中扮演着重要的角色，但在某些情况下，使用Socket可以提供更好的性能和更大的灵活性。再者，Binder当初并不成熟，团队成员对于进程间通讯更倾向于用Socket，后面为了做了很多优化，才使得Binder通讯变得成熟稳定。

## 5.每个App都会将系统的资源，系统的类都加载一遍吗
zygote进程的作用：
1.创建一个Service端的Socket，开启一个ServerSocket实现和别的进程通信。
2.加载系统类，系统资源。
3.启动System Server进程

Zygote进程预加载系统资源后，然后通过它孵化出其他的虚拟机进程，进而共享虚拟机内存和框架层资源（共享内存），这样大幅度提高应用程序的启动和运行速度。

## 6.PMS 是干什么的，你是怎么理解PMS
包管理，包解析，结果缓存，提供查询接口。

遍历/data/app的文件夹
解压apk文件
dom解析AndroidManifest.xml文件。

## 7.为什么会有AMS AMS的作用
查询PMS
反射生成对象
管理Activity生命周期
AMS缓存中心：ActivityThread

## 8.AMS如何管理Activity，探探AMS的执行原理
Activity在应用端由ActivityClientRecord负责描述其生命周期的过程与状态，但最终这些过程与状态是由ActivityManagerService(以下简称AMS)来管理和控制的

BroadcastRecord：描述了应用进程的BroadcastReceiver，由BroadcastQueue负责管理。
ServiceRecord：描述了Service服务组件，由ActiveServices负责管理。
ContentProviderRecord：描述ContentProvider内容提供者，由ProviderMap管理。
ActivityRecord：用于描述Activity，由ActivityStackSupervisor进行管理。
