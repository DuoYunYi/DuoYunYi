
# 一、Handler

[【Android Framework系列】第1章 Handler消息传递机制_android binder通信,handler-CSDN博客](https://blog.csdn.net/u010687761/article/details/130915378?spm=1001.2014.3001.5502)

https://github.com/huangruiLearn/hrl_android_notes/blob/master/1.android/5.Handler/Handler.md

## 1.简介

消息传递机制，Handler主要用于同进程的**线程间**通信。而Binder/Socket用于**进程间**通信。
## 2.Handler运行机制

 主要会涉及到四个类：Handler、Looper、Message、MessageQueue

一句话理解：**Message通过Handler添加到当前线程的MessageQueue队列，当前线程的Looper不断轮询获取Message并通知对应Handler处理消息。**
 
```
Handler：消息处理器，通过obtainMessage()生成消息和handleMessage()处理消息；

Looper：循环器，用于取出消息，每个线程只能够一个Looper，管理MessageQueue，不断从中将Message分发并通知Handler处理消息；

Message：消息类，Handler的接收和处理对象通过obtain()方法可以生成一个Message对象，每个Message对象必须有一个Handler对象target，要执行该消息时通过target找到对应Handler执行

MessageQueue：消息队列，用于存储消息，通过enqueueMessage()放入消息，next()取出消息。
```

## 3.Handler相关问答

### （1）Handler实现原理
Handler发送消息时调用MessageQueue的enqueueMessage插入一条信息到MessageQueue,Looper不断轮询调用MeaasgaQueue的next方法 如果发现message就调用handler的dispatchMessage，dispatchMessage被成功调用，接着调用handlerMessage()。
### （2）可以在子线程直接new一个Handler吗？为什么主线程可以
不能，因为Handler 的构造方法中，会通过Looper.myLooper()获取looper对象，如果为空，则抛出异常，  
主线程则因为已在入口处ActivityThread的main方法中通过 Looper.prepareMainLooper()获取到这个对象，  
并通过 Looper.loop()开启循环，在子线程中若要使用handler，**可先通过Loop.prepare获取到looper对象，并使用Looper.loop()开启循环**
### （3）为什么系统不建议在子线程访问UI？
（为什么不能在子线程更新UI？）
在某些情况下，在子线程中是可以更新UI的。但是在ViewRootImpl中对UI操作进行了
checkThread，但是我们在OnCreate和onResume中可以使用子线程更新UI，由于我们在
ActivityThread中的performResumeActivity方法中通过addView创建了ViewRootImpl，这个行为是在onResume之后调用的，所以在OnCreate和onResume可以进行更新UI。
但是我们不能在子线程中更新UI，因为UI更新本身是线程不安全的，如果添加了耗时操作之后，一旦ViewRootImpl被创建将会抛出异常。一旦在子线程中更新UI，容易产生并发问题。
### （4） Handler内存泄漏问题及解决方案
内部类持有外部类的引用导致了内存泄漏，如果Activity退出的时候，MessageQueue中还有一个Message没有执行，这个Message持有了Handler的引用，而Handler持有了Activity的引用，导致Activity无法被回收，导致内存泄漏。使用static关键字修饰，在onDestory的时候将消息清除。
简单理解：
当Handler为非静态内部类时，其持有外部类Actviity对象，所以导致Looper->MessageQueue->Message->Handler->Activity，这个引用链中Message如果还在MessageQueue中等待执行，则会导致Activity一直无法被释放和回收。

导致的原因：
因为Looper需要循环，所以一个线程只有一个Looper，但一个线程中可有多个Handler，MessageQueue中消息Message执行时不知道要通知哪个Handler执行任务，
所以在Message创建时中存入了Handler对象target用于回调执行的消息。如果Handler是Activity这种短生命周期对象的非静态内部类时，则会让创建出来的Handler对象持有该外部类Activity的引用，Message还在队列中导致引用着Handler，而非静态内部类Handler引用外部类Activity导致Activity无法被回收，最终导致内存泄漏。

解决办法：
1.Handler不能是Activity这种短生命周期的对象类的内部类；
2.在Activity销毁时，将创建的Handler中的消息队列清空并结束所有任务
### （5）一个Thread可以有几个Looper？几个Handler？
一个线程可以有多个Handler,只有一个Looper对象,只有一个MessageQueue对象。Looper.prepare()函数中知道，。在Looper的prepare方法中创建了Looper对象，并放入到ThreadLocal中，并通过ThreadLocal来获取looper的对象, ThreadLocal的内部维护了一个ThreadLocalMap类,ThreadLocalMap是以当前thread做为key的,因此可以得知，一个线程最多只能有一个Looper对象， 在Looper的构造方法中创建了MessageQueue对象，并赋值给mQueue字段。因为Looper对象只有一个，那么Messagequeue对象肯定只有一个。
### （6）Message可以如何创建？哪种效果更好，为什么？
Message.obtain来创建Message。通过这种方式创建的Message会被存放在一个大小为50的复用池中，这样会复用之前的Message的内存，不会频繁的创建对象，导致内存抖动。
### （7）Message对象创建的方式有哪些 & 区别
**Message.obtain()怎么维护消息池的**  
1.Message msg = **new Message();**  
每次需要Message对象的时候都创建一个新的对象，每次都要去堆内存开辟对象存储空间 
2.Message msg = **Message.obtain();**  
obtainMessage能避免重复Message创建对象。它先判断消息池是不是为空，如果非空的话就从消息池表头的Message取走,再把表头指向 next。  
如果消息池为空的话说明还没有Message被放进去，那么就new出来一个Message对象。消息池使用 Message 链表结构实现，消息池默认最大值 50。消息在loop中被handler分发消费之后会执行回收的操作，将该消息内部数据清空并添加到消息链表的表头。  
3.Message msg = **handler.obtainMessage();** 其内部也是调用的obtain()方法
### （8）Handler 有哪些发送消息的方法  
sendMessage(Message msg)  
sendMessageDelayed(Message msg, long uptimeMillis)  
post(Runnable r)  
postDelayed(Runnable r, long uptimeMillis)  
sendMessageAtTime(Message msg,long when)
### （9）Handler的post与sendMessage的区别

| 特性         | post(Runnable)    | sendMessage(Message)          |
| ---------- | ----------------- | ----------------------------- |
| **参数类型**   | Runnable 对象       | Message 对象                    |
| **使用复杂度**  | 简单直接              | 相对复杂                          |
| **数据传递能力** | 有限（通过final变量）     | 强大（可使用Message的arg1,arg2,obj等） |
| **代码可读性**  | 较高（逻辑集中）          | 较低（逻辑分散）                      |
| **性能开销**   | 略高（需创建Runnable对象） | 略低（可复用Message对象）              |
| **适用场景**   | 简单UI更新、短小任务       | 复杂消息传递、需要携带数据                 |

### （10）使用Hanlder的postDealy()后消息队列会发生什么变化？
Handler发送消息到消息队列，消息队列是一个时间优先级队列，内部是一个单向链表。发动postDelay之后会将该消息进行时间排序存放到消息队列中。
### （11）MessageQueue是什么数据结构
内部存储结构并不是真正的队列，而是采用**单链表**的数据结构来存储消息列表  
这点和传统的队列有点不一样，主要区别在于Android的这个队列中的消息是按照时间先后顺序来存储的，时间较早的消息，越靠近队头。 当然，我们也可以理解成，它是先进先出的，只是这里的先依据的不是谁先入队，而是消息待发送的时间
### （12）HandlerThread是什么 & 好处 &原理 & 使用场景
HandlerThread本质上是一个线程类，它继承了Thread； HandlerThread有自己的内部Looper对象，通过Looper.loop()进行looper循环；  
通过获取HandlerThread的looper对象传递给Handler对象，然后在handleMessage()方法中执行异步任务；

优势:  
1.将loop运行在子线程中处理,减轻了主线程的压力,使主线程更流畅,有自己的消息队列,不会干扰UI线程  
2.串行执行,开启一个线程起到多个线程的作用

劣势:  
1.由于每一个任务队列逐步执行,一旦队列耗时过长,消息延时  
2.对于IO等操作,线程等待,不能并发

我们可以使用HandlerThread处理本地IO读写操作（数据库，文件），因为本地IO操作大多数的耗时属于毫秒级别，对于单线程 + 异步队列的形式 不会产生较大的阻塞
### （13）主线程中Looper的轮询死循环为何没有阻塞主线程？
Looper轮询是死循环，但是当没有消息的时候他会block（阻塞），ANR是当我们处理点击事件的时候5s内没有响应，我们在处理点击事件的时候也是用的Handler，所以一定会有消息执行，并且ANR也会发送Handler消息，所以不会阻塞主线程。Looper是通过Linux系统的epoll实现的阻塞式等待消息执行（有消息执行无消息阻塞），而ANR是消息处理耗时太长导致无法执行剩下需要被执行的消息触发了ANR。
### （14）Handler是如何完成子线程和主线程通信的？
在主线程中创建Handler，在子线程中发送消息，放入到MessageQueue中,通过Looper.loop取出消息进行执行handleMessage，由于looper我们是在主线程初始化的，在初始化looper的时候会创建消息队列，所以消息是在主线程被执行的。
### （15）关于ThreadLocal，谈谈你的理解？
ThreadLocal实例进程内只有一个（静态实例），但其内部的set和get方法是获取的当前线程的ThreadLocalMap对象，ThreadLocalMap是每个线程有一个单独的内存空间，不共享，ThreadLocal在set的时候会将数据存入对应线程的ThreadLocalMap中，key=ThreadLocal，value=值
### （16） 既然可以存在多个Handler往MessageQueue中添加数据(发消息是各个handler可能处于不同线程)，那他内部是如何确保线程安全的？
在添加数据和执行next的时候都加了this锁，这样可以保证添加的位置是正确的，获取的也会是最前面的。
### （17） 关于IntentService，谈谈你的理解
HandlerThread+Service实现，可以实现Service在子线程中执行耗时操作，并且执行完耗时操作时候会将自己stop。
### （18）handler 主线程阻塞了怎么办，阻塞怎么唤醒？
Android系统事件驱动系统，loop循环处理事件，如果不循环程序就结束了

# 二、Binder

[【Android Framework系列】第2章 Binder机制大全_android binder-CSDN博客](https://blog.csdn.net/u010687761/article/details/131057192?spm=1001.2014.3001.5502)

https://blog.csdn.net/carson_ho/article/details/73560642

## 1.简介

1、进程是什么？  
它是系统进行资源分配和调度的一个独立单位,也就是说进程是可以独立运行的一段程序。  
2、线程又是什么？  
线程进程的一个实体，是CPU调度和分派的基本单位，他是比进程更小的能独立运行的基本单位,线程自己基本上不拥有系统资源。在运行时，只是暂用一些计数器、寄存器和栈 。

- 进程有不同的代码和数据空间，而多个线程则共享数据空间，每个线程有自己的执行堆栈和程序计数器为其执行上下文。  
- 进程间相互独立，同一进程的各线程间共享。  
- 进程间通信IPC，线程间可以直接读写进程数据段（如全局变量）来进行通信——需要进程同步和互斥手段的辅助，以保证数据的一致性。

IPC:Inter-Process Communication 进程间通信

Binder是Android中主要的**跨进程**通信方式。

Android系统中，每个应用程序是由Android的Activity，Service，BroadCast，ContentProvider这四剑客中一个或多个组合而成，这四剑客所涉及的**多进程间的通信底层都是依赖于BinderIPC机制**。例如当进程A中的Activity要向进程B中的Service通信，这便需要依赖于BinderIPC。

Zygote通信便是采用socket。

## 2.为什么要使用Binder

传统Linux进程间通信方式：管道、信号量、socket、共享内存，**而Binder是Android系统独有的通讯方式**

```
Binder：只需要拷贝一次，基于C/S架构，易用性高，系统为每个APP分配UID同时支持实名和匿名更安全
共享内存：无需拷贝，控制复杂，易用性差，依赖上层协议，访问接入点是开放的不安全

Socket：需要拷贝两次，基于C/S架构，作为一款通用接口，其传输效率低，开销大，以来上层协议，访问接入点是开放的，不安全


管道：需要拷贝两次，非C/S架构，是一对一的通讯模型，把一个程序的输出直接链接另一个程序的输入。
Linux下的管道主要分两种：无名管道(pipe)和有名管道(fifo)。无名管道只能用于具有亲缘关系的进程之间的通信(也就是父子进程或兄弟进程之间)，是以半双工的一个通信方式，速度较慢，容量有限；有名管道是对无名管道的一种改进，可以让两个互补相干的进程之间进行通信。并且该管道在文件系统中可见，不过大小一直为0。

信号量：与其他的进程间通信方式不太相同，它主要提供对进程间共享资源的访问机制，进程会根据它判定是否能够访问某些共享资源，同时进程也可以修改该标志。除了用于访问控制外，还可以用于进程同步，主要是用来解决进程线程之间同步与互斥问题的一种通信机制。
```


Android系统需要一种**高效率、安全性高**的方式，因此Binder最合适不过。**Binder只需要拷贝一次，效率仅次于共享内存，而且采用传统的C/S结构**

![[Pasted image 20250611105208.png]]


## 3.什么是Binder

- 从进程间通信的角度看，Binder 是一种进程间通信的机制；

- 从 Server 进程的角度看，Binder 指的是 Server 中的 Binder 实体对象(Binder类 IBinder)；

- 从 Client 进程的角度看，Binder 指的是对 Binder 代理对象，是 Binder 实体对象的一个远程代理

- 从传输过程的角度看，Binder 是一个可以跨进程传输的对象；Binder 驱动会自动完成代理对象和本地对象之间的转换。

- 从Android Framework角度来说，Binder是ServiceManager连接各种Manager和相应ManagerService的桥梁 Binder跨进程通信机制：基于C/S架构，由Client、Server、ServerManager和Binder驱动组成。

进程空间分为用户空间和内核空间。用户空间不可以进行数据交互；内核空间可以进行数据交互，所有进程共用一个内核空间

Client、Server、ServiceManager均在用户空间中实现，而Binder驱动程序则是在内核空间中实现的；

## 4.Binder的原理

**Binder Driver 如何在内核空间中做到一次拷贝的**

进程空间分为用户空间和内核空间。用户空间不可以进行数据交互；内核空间可以进行数据交互，所有进程共用一个内核空间。

应用程序不能直接操作设备硬件地址,如果用户空间需要读取磁盘的文件， 如果不采用内存映射， 需要两次拷贝（磁盘-->内核空间-->用户空间）；

内存映射将用户空间的一块内存区域映射到内核空间。映射关系建立后，内核空间对这段区域的修改也能直接反应到用户空间,少了一次拷贝。

Binder 驱动使用 mmap() 在内核空间创建数据接收的缓存空间。 mmap(NULL, MAP_SIZE, PROT_READ, MAP_PRIVATE, fd, 0)的返回值是内核空间映射在用户空间的地址

1.Binder 驱动在内核空间创建一个数据接收缓存区。

2.在内核空间开辟一块内核缓存区，建立内核缓存区和内核空间的数据接收缓存区之间的映射关系，以及内核中数据接收缓存区和接收进程用户空间地址的映射关系。

3.发送方进程通过系统调用 copy_from_user() 将数据 copy 到内核空间的内核缓存区，由于内核缓存区和接收进程的用户空间存在内存映射，因此也就相当于把数据发送到了接收进程的用户空间，这样便完成了一次进程间的通信。

![[binder3.png]]


## 5.Binder数据传输的具体过程

**系统层面*

注册服务:
服务进程向Binder进程发起服务注册
Binder驱动将注册请求转发给ServiceManager进程
ServiceManager进程添加这个服务进程

获取服务:
用户进程向Binder驱动发起获取服务的请求，传递要获取的服务名称Binder驱动
将该请求转发给ServiceManager进程
ServiceManager进程查到到用户进程需要的服务进程信息最后
通过Binder驱动将上述服务信息返回个用户进程

使用服务:
1.Binder通过内存映射建立数据缓存区
2.根据ServiceManager查到的服务的进程和数据缓存区 , 数据缓存区和client进程的内存缓存区建立映射
3.client调用copy_from_user数据到内存缓存区
4.收到binder启动后服务进程根据用户进程要求调用目标方法
5.服务进程将目标方法的结果返回给用户进程

![[binder4.png]]

**具体代码层面**
1、服务端中的Service给客户端提供Binder对象
2、客户端通过AIDL接口中的asInterface()将这个Binder对象转换为代理Proxy并通过它发起RPC请求
3、client进程的请求数据data通过代理binder对象的transact方法，发送到内核空间，当前线程被挂起
4、server进程收到binder驱动通知， onTransact(在线程池中进行数据反序列化&调用目标方法)处理客户端请求，并将结果写入reply
5、Binder驱动将server进程的目标方法执行结果，拷贝到client进程的内核空间
6、Binder驱动通知client进程，之前挂起的线程被唤醒，并收到返回结果

![[Pasted image 20250724163922.png]]


## 6.Binder其他相关知识
### （1）Binder框架中ServiceManager的作用

ServiceManager使得客户端可以获取服务端binder实例对象的引用

### （2）使用 Binder 传输数据的最大限制是多少，被占满后会导致什么问题

因为Binder本身就是为了进程间频繁而灵活的通信所设计的，并不是为了拷贝大数据而使用的。比如在Activity之间传输BitMap的时候，如果Bitmap过大，就会引起问题，比如崩溃等，这其实就跟Binder传输数据大小的限制有关系

mmap函数会为Binder数据传递映射一块连续的虚拟地址，这块虚拟内存空间其实是有大小限制。

普通的由Zygote孵化而来的用户进程，所映射的Binder内存大小是不到1M的，准确说是 (1 * 1024 * 1024) - (4096 * 2)

```
#define BINDER_VM_SIZE ((1*1024*1024) - (4096 *2))
```

特殊的进程ServiceManager进程，它为自己申请的Binder内核空间是**128K**，这个同ServiceManager的用途是分不开的，ServcieManager主要面向系统Service，只是简单的提供一些addServcie，getService的功能，不涉及多大的数据传输，因此不需要申请多大的内存：

```
 bs = binder_open(128*1024);
```

当服务端的内存缓冲区被Binder进程占用满后，Binder驱动不会再处理binder调用并在c++层抛出DeadObjectException到binder客户端

### （3）Binder 驱动加载过程中有哪些重要的步骤
从 Java 层来看就像访问本地接口一样，客户端基于 BinderProxy 服务端基于 IBinder 对象 。

在Native层有一套完整的binder通信的C/S架构，Bpinder作为客户端，BBinder作为服务端。基于naive层的Binder框架，Java也有一套镜像功能的binder C/S架构，通过JNI技术，与native层的binder对应，Java层的binder功能最终都是交给native的binder来完成。

从内核看跨进程通信的原理最终是要基于内核的，所以最会会涉及到 binder_open 、binder_mmap 和 binder_ioctl这三种系统调用。
![[Pasted image 20250724164341.png]]

### （4）Activity的bindService流程

1、Activity调用bindService：通过Binder通知ActivityManagerService，要启动哪个Service

2、ActivityManagerService创建ServiceRecord，并利用ApplicationThreadProxy回调，通知APP新建并启动Service启动起来

3、ActivityManagerService把Service启动起来后，继续通过ApplicationThreadProxy，通知APP，bindService，其实就是让Service返回一个Binder对象给ActivityManagerService，以便AMS传递给Client

4、ActivityManagerService把从Service处得到这个Binder对象传给Activity，这里是通过IServiceConnection binder实现。

5、Activity被唤醒后通过Binder Stub的asInterface函数将Binder转换为代理Proxy，完成业务代理的转换，之后就能利用Proxy进行通信了。


 ### （5）进程空间划分

    一个进程空间分为 用户空间 & 内核空间（Kernel），即把进程内 用户 & 内核 隔离开来
    二者区别：
        进程间，用户空间的数据不可共享，所以用户空间 = 不可共享空间
        进程间，内核空间的数据可共享，所以内核空间 = 可共享空间

    所有进程共用1个内核空间

进程内 用户空间 & 内核空间 进行交互 需通过 **系统调用**，主要通过函数：
1. copy_from_user（）：将用户空间的数据拷贝到内核空间
2. copy_to_user（）：将内核空间的数据拷贝到用户空间
![[Pasted image 20250717161545.png]]


# 三、Broadcast
