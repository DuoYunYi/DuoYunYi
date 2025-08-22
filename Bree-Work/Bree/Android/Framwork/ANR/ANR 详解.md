# 一、ANR原因

**（Application Not Responding）**

## 应用层导致ANR（耗时操作）
1、函数阻塞：如死循环、主线程IO、处理大数据  
2、锁出错：主线程等待子线程的锁  
3、内存紧张：系统分配给一个应用的内存是有上限的，长期处于内存紧张，会导致频繁内存交换，进而导致应用的一些操作超时

## 系统导致ANR
1、CPU被抢占：一般来说，前台在玩游戏，可能会导致你的后台广播被抢占CPU  
2、系统服务无法及时响应：比如获取系统联系人等，系统的服务都是[Binder机制](https://so.csdn.net/so/search?q=Binder%E6%9C%BA%E5%88%B6&spm=1001.2101.3001.7020)，服务能力也是有限的，有可能系统服务长时间不响应导致ANR  
3、其他应用占用的大量内存

## 超时时间

- Service Timeout：前台服务在20s内未执行完成；
    
- BroadcastQueue Timeout：前台广播在10s内未执行完成
    
- ContentProvider Timeout：内容提供者在publish过超时10s;
    
- InputDispatching Timeout：输入事件分发超时5s，包括按键和触摸事件。


（1）KeyDispatchTimeout（常见）
input事件在5S内没有处理完成发生了ANR。
logcat日志关键字：Input event dispatching timed out

（2）BroadcastTimeout
前台Broadcast：onReceiver在10S内没有处理完成发生ANR。
后台Broadcast：onReceiver在60s内没有处理完成发生ANR。
logcat日志关键字：Timeout of broadcast BroadcastRecord

（3）ServiceTimeout
前台Service：onCreate，onStart，onBind等生命周期在20s内没有处理完成发生ANR。
后台Service：onCreate，onStart，onBind等生命周期在200s内没有处理完成发生ANR
logcat日志关键字：Timeout executing service

（4）ContentProviderTimeout
ContentProvider 在10S内没有处理完成发生ANR。
logcat日志关键字：timeout publishing content providers


# 二、日志分析

## 1.CPU负载

```
CPU usage from 28360ms to 0ms ago (2024-02-29 13:10:31.735 to 2024-02-29 13:11:00.095):
  90% 1377/system_server: 42% user + 48% kernel / faults: 72533 minor 370 major
  22% 295/logd: 5.4% user + 16% kernel / faults: 4061 minor
  20% 789/surfaceflinger: 8.6% user + 12% kernel / faults: 1413 minor
........省略N行.....
83% TOTAL: 32% user + 37% kernel + 4.7% iowait + 7.5% irq + 1.5% softirq

```
如上所示：

    第一行：表明负载信息抓取在ANR发生之后的0~28360ms。同时也指明了ANR的时间点：2024-02-29 13:11:00.095
    中间部分：各个进程占用的CPU的详细情况
    最后一行：各个进程合计占用的CPU信息。
    
        a. user:用户态,kernel:内核态
        b. faults:内存缺页，minor——轻微的，major——重度，需要从磁盘拿数据
        c. iowait:IO使用（等待）占比
        d. irq:硬中断，softirq:软中断

    iowait占比很高，意味着有很大可能，是io耗时导致ANR，具体进一步查看有没有进程faults major比较多。

    单进程CPU的负载并不是以100%为上限，而是有几个核，就有百分之几百，如4核上限为400%。

## 2.内存

```
Total number of allocations 476778　　//进程创建到现在一共创建了多少对象

Total bytes allocated 52MB　//进程创建到现在一共申请了多少内存

Total bytes freed 52MB　　　//进程创建到现在一共释放了多少内存

Free memory 777KB　　　 //不扩展堆的情况下可用的内存

Free memory until GC 777KB　　//GC前的可用内存

Free memory until OOME 383MB　　//OOM之前的可用内存

Total memory 28MB//当前总内存（已用+可用）

Max memory 384MB  //进程最多能申请的内存
```

- 从含义可以得出结论：Free memory until OOME 的值很小的时候，已经处于内存紧张状态。应用可能是占用了过多内存。

**ps:如果ANR时间点前后，日志里有打印onTrimMemory，也可以作为内存紧张的一个参考判断**

## 3.堆栈信息

堆栈信息是最重要的一个信息，展示了ANR发生的进程当前所有线程的状态。

```
suspend all histogram:  Sum: 2.834s 99% C.I. 5.738us-7145.919us Avg: 607.155us Max: 41543us
DALVIK THREADS (248):
"main" prio=5 tid=1 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x74b17080 self=0x7bb7a14c00
  | sysTid=2080 nice=-2 cgrp=default sched=0/0 handle=0x7c3e82b548
  | state=S schedstat=( 757205342094 583547320723 2145008 ) utm=52002 stm=23718 core=5 HZ=100
  | stack=0x7fdc995000-0x7fdc997000 stackSize=8MB
  | held mutexes=
  kernel: __switch_to+0xb0/0xbc
  kernel: SyS_epoll_wait+0x288/0x364
  kernel: SyS_epoll_pwait+0xb0/0x124
  kernel: cpu_switch_to+0x38c/0x2258
  native: #00 pc 000000000007cd8c  /system/lib64/libc.so (__epoll_pwait+8)
  native: #01 pc 0000000000014d48  /system/lib64/libutils.so (android::Looper::pollInner(int)+148)
  native: #02 pc 0000000000014c18  /system/lib64/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+60)
  native: #03 pc 0000000000127474  /system/lib64/libandroid_runtime.so (android::android_os_MessageQueue_nativePollOnce(_JNIEnv*, _jobject*, long, int)+44)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:330)
  at android.os.Looper.loop(Looper.java:169)
  at com.android.server.SystemServer.run(SystemServer.java:508)
  at com.android.server.SystemServer.main(SystemServer.java:340)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:536)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:856)
   
  ........省略N行.....
   
  "OkHttp ConnectionPool" daemon prio=5 tid=251 TimedWaiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13daea90 self=0x7bad32b400
  | sysTid=29998 nice=0 cgrp=default sched=0/0 handle=0x7b7d2614f0
  | state=S schedstat=( 951407 137448 11 ) utm=0 stm=0 core=3 HZ=100
  | stack=0x7b7d15e000-0x7b7d160000 stackSize=1041KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x05e5732e> (a com.android.okhttp.ConnectionPool)
  at com.android.okhttp.ConnectionPool$1.run(ConnectionPool.java:103)
  - locked <0x05e5732e> (a com.android.okhttp.ConnectionPool)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

```


如上日志所示，本文截图了两个线程信息，一个是主线程main，它的状态是native。
另一个是OkHttp ConnectionPool，它的状态是TimeWaiting。众所周知，教科书上说线程状态有5种：新建、就绪、执行、阻塞、死亡。而Java中的线程状态有6种，6种状态都定义在：java.lang.Thread.State中

![[Pasted image 20250717202224.png]]
![[Pasted image 20250717203156.png]]

由此可知，main函数的native状态是正在执行JNI函数。堆栈信息是我们分析ANR的第一个重要的信息，一般来说：  
main线程处于 BLOCK、WAITING、TIMEWAITING状态，那基本上是函数阻塞导致ANR；

**如果main线程无异常，则应该排查CPU负载和内存环境。**

## 4.Trace日志

```
----- pid 23326 at 2023-02-21 00:15:26 -----
Cmd line: com.demo
Build fingerprint: 'google/android_x86/x86:7.1.2/N2G48C/N975FXXU1ASGO:user/release-keys'
ABI: 'x86'                                 //x86一般为模拟器
Build type: optimized
Zygote loaded classes=4380 post zygote classes=9911
Intern table: 48884 strong; 554 weak
JNI: CheckJNI is off; globals=1079 (plus 1467 weak)
Libraries: ...
Heap: 17% free, 43MB/52MB; 674957 objects  //已分配堆内存大小52MB，其中已用43MB，剩下17%空余，共分配了674957个对象
Total time spent in GC: 6.414s             //GC用过了多少时间
Mean GC size throughput: 244MB/s
Mean GC object throughput: 6.31642e+06 objects/s
Total number of allocations 41189203       //进程创建到现在一共创建了多少对象
Total bytes allocated 1GB                  //进程创建到现在一共申请了1GB内存
Total bytes freed 1GB                      //进程创建到现在一共释放了1GB内存 
Free memory 8MB                            //不扩展堆堆情况下可用的内存
Free memory until GC 8MB                   //GC前的可用内存
Free memory until OOME 212MB               //OOM之前的可用内存，这个值很小，说明内存紧张
Total memory 52MB                          //当前总内存（已用+可用）
Max memory 256MB                           //进程最多能申请的内存
Zygote space size 1552KB
Total mutator paused time: 177.563ms
Total time waiting for GC to complete: 89.366ms
Total GC count: 130                        //GC次数
Total GC time: 6.414s                      //GC时长
Total blocking GC count: 1                 //GC被block次数以及时长
Total blocking GC time: 100.948ms
suspend all histogram:	Sum: 65.115ms 99% C.I. 5.440us-15646.719us Avg: 478.786us Max: 35628us
DALVIK THREADS (65):                       //当前进程共有65个线程

"Signal Catcher" daemon prio=5 tid=3 Runnable
  | group="system" sCount=0 dsCount=0 obj=0x12c3a310 self=0xb6ed7e00
  | sysTid=23332 nice=0 cgrp=default sched=0/0 handle=0xc2ba6910
  | state=R schedstat=( 63328804 15730865 32 ) utm=3 stm=2 core=3 HZ=100
  | stack=0xc2aaa000-0xc2aac000 stackSize=1014KB
  | held mutexes= "mutator lock"(shared held)
  
// Signal Catcher：线程名  daemon: 守护线程  prio：线程优先级  tid：线程内部id  Runnable: 线程状态
// group: 线程所属的线程组  sCount：线程挂起次数  dsCount：用于调试的线程挂起次数  obj：当前线程关联的Java线程对象  self：当前线程地址
// sysTid：线程真正意义上的tid  nice：调度优先级，值越小优先级越高  cgrp: 进程所属的进程调度组  sched: 调度策略  handle：函数处理地址
// state：线程状态  schedstat：CPU调度时间统计，依次表示cpu运行时间、RQ队列等待时间、cpu调度切换次数  utm: 用户态的cpu时间  stm：内核态的cpu时间  core：该线程的最后运行所在核  HZ：时钟频率
// stack：线程栈的地址空间  stackSize：栈的大小
// mutexes：所持有 mutex 类型，有独占锁（exclusive）和 共享锁（shared）两类

```

## 5.Bugreport

Bugreport日志比较冗长，包含设备的各种信息，对于ANR来说，我们只要过滤ANR需要的关键字即可  

（1）搜索am_anr

通过当前关键字可以找到最终发送ANR的罪魁祸首

```
03-07 22:01:47.632  1000  1675 15617 I am_anr  : [0,6267,com.demo,953695814,executing service com.demo/com.demo.hensen.MyService]

```

（2）搜索ANR in  
通过当前关键字可以找到最终发送ANR的罪魁祸首

```
E ActivityManager: ANR in com.demo.hensen.MyService
E ActivityManager: PID: 3036
E ActivityManager: Reason: executing service com.demo/com.demo.hensen.MyService

```
# 三、案例分析

### 1.主线程无卡顿，处于正常状态堆栈

```
"main" prio=5 tid=1 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x74b38080 self=0x7ad9014c00
  | sysTid=23081 nice=0 cgrp=default sched=0/0 handle=0x7b5fdc5548
  | state=S schedstat=( 284838633 166738594 505 ) utm=21 stm=7 core=1 HZ=100
  | stack=0x7fc95da000-0x7fc95dc000 stackSize=8MB
  | held mutexes=
  kernel: __switch_to+0xb0/0xbc
  kernel: SyS_epoll_wait+0x288/0x364
  kernel: SyS_epoll_pwait+0xb0/0x124
  kernel: cpu_switch_to+0x38c/0x2258
  native: #00 pc 000000000007cd8c  /system/lib64/libc.so (__epoll_pwait+8)
  native: #01 pc 0000000000014d48  /system/lib64/libutils.so (android::Looper::pollInner(int)+148)
  native: #02 pc 0000000000014c18  /system/lib64/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+60)
  native: #03 pc 00000000001275f4  /system/lib64/libandroid_runtime.so (android::android_os_MessageQueue_nativePollOnce(_JNIEnv*, _jobject*, long, int)+44)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:330)
  at android.os.Looper.loop(Looper.java:169)
  at android.app.ActivityThread.main(ActivityThread.java:7073)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:536)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:876)

```

上述主线程堆栈就是一个很正常的空闲堆栈，表明主线程正在等待新的消息。
如果ANR日志里主线程是这样一个状态，那可能有两个原因：

    该ANR是CPU抢占或内存紧张等其他因素引起
    这份ANR日志抓取的时候，主线程已经恢复正常

遇到这种空闲堆栈，可以按照第2章的方法去分析CPU、内存的情况。其次可以关注抓取日志的时间和ANR发生的时间是否相隔过久，时间过久这个堆栈就没有分析意义了。

### 2.主线程执行耗时操作

```
"main" prio=5 tid=1 Runnable
  | group="main" sCount=0 dsCount=0 flags=0 obj=0x72deb848 self=0x7748c10800
  | sysTid=8968 nice=-10 cgrp=default sched=0/0 handle=0x77cfa75ed0
  | state=R schedstat=( 24783612979 48520902 756 ) utm=2473 stm=5 core=5 HZ=100
  | stack=0x7fce68b000-0x7fce68d000 stackSize=8192KB
  | held mutexes= "mutator lock"(shared held)
  at com.example.test.MainActivity$onCreate$2.onClick(MainActivity.kt:20)——关键行！！！
  at android.view.View.performClick(View.java:7187)
  at android.view.View.performClickInternal(View.java:7164)
  at android.view.View.access$3500(View.java:813)
  at android.view.View$PerformClick.run(View.java:27640)
  at android.os.Handler.handleCallback(Handler.java:883)
  at android.os.Handler.dispatchMessage(Handler.java:100)
  at android.os.Looper.loop(Looper.java:230)
  at android.app.ActivityThread.main(ActivityThread.java:7725)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:526)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1034)

```

上述日志表明，主线程正处于执行状态，看堆栈信息可知不是处于空闲状态，发生ANR是因为一处click监听函数里执行了耗时操作。

### 3.主线程被锁阻塞

```
"main" prio=5 tid=1 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x72deb848 self=0x7748c10800
  | sysTid=22838 nice=-10 cgrp=default sched=0/0 handle=0x77cfa75ed0
  | state=S schedstat=( 390366023 28399376 279 ) utm=34 stm=5 core=1 HZ=100
  | stack=0x7fce68b000-0x7fce68d000 stackSize=8192KB
  | held mutexes=
  at com.example.test.MainActivity$onCreate$1.onClick(MainActivity.kt:15)
  - waiting to lock <0x01aed1da> (a java.lang.Object) held by thread 3 ——————关键行！！！
  at android.view.View.performClick(View.java:7187)
  at android.view.View.performClickInternal(View.java:7164)
  at android.view.View.access$3500(View.java:813)
  at android.view.View$PerformClick.run(View.java:27640)
  at android.os.Handler.handleCallback(Handler.java:883)
  at android.os.Handler.dispatchMessage(Handler.java:100)
  at android.os.Looper.loop(Looper.java:230)
  at android.app.ActivityThread.main(ActivityThread.java:7725)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:526)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1034)
   
  ........省略N行.....
   
  "WQW TEST" prio=5 tid=3 TimeWating
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x12c44230 self=0x772f0ec000
  | sysTid=22938 nice=0 cgrp=default sched=0/0 handle=0x77391fbd50
  | state=S schedstat=( 274896 0 1 ) utm=0 stm=0 core=1 HZ=100
  | stack=0x77390f9000-0x77390fb000 stackSize=1039KB
  | held mutexes=
  at java.lang.Thread.sleep(Native method)
  - sleeping on <0x043831a6> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:440)
  - locked <0x043831a6> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:356)
  at com.example.test.MainActivity$onCreate$2$thread$1.run(MainActivity.kt:22)
  - locked <0x01aed1da> (a java.lang.Object)————————————————————关键行！！！
  at java.lang.Thread.run(Thread.java:919)

```

这是一个典型的主线程被锁阻塞的例子；
其中等待的锁是<0x01aed1da>，这个锁的持有者是线程 3。进一步搜索 “tid=3” 找到线程3， 发现它正在TimeWating。
那么ANR的原因找到了：线程3持有了一把锁，并且自身长时间不释放，主线程等待这把锁发生超时。在线上环境中，常见因锁而ANR的场景是SharePreference写入。

## 4.CPU被抢占

```
CPU usage from 0ms to 10625ms later (2020-03-09 14:38:31.633 to 2020-03-09 14:38:42.257):
  543% 2045/com.alibaba.android.rimet: 54% user + 89% kernel / faults: 4608 minor 1 major ————关键行！！！
  99% 674/android.hardware.camera.provider@2.4-service: 81% user + 18% kernel / faults: 403 minor
  24% 32589/com.wang.test: 22% user + 1.4% kernel / faults: 7432 minor 1 major
  ........省略N行.....

```

如上日志，第二行是钉钉的进程，占据CPU高达543%，抢占了大部分CPU资源，因而导致发生ANR。

## 5.内存紧张导致ANR

如果有一份日志，CPU和堆栈都很正常（不贴出来了），仍旧发生ANR，考虑是内存紧张。
从CPU第一行信息可以发现，ANR的时间点是2020-10-31 22:38:58.468—CPU usage from 0ms to 21752ms later (2020-10-31 22:38:58.468 to 2020-10-31 22:39:20.220)
接着去logcat里搜索am_meminfo， 这个没有搜索到。再次搜索onTrimMemory，果然发现了很多条记录；

```
10-31 22:37:19.749 20733 20733 E Runtime : onTrimMemory level:80,pid:com.xxx.xxx:Launcher0
10-31 22:37:33.458 20733 20733 E Runtime : onTrimMemory level:80,pid:com.xxx.xxx:Launcher0
10-31 22:38:00.153 20733 20733 E Runtime : onTrimMemory level:80,pid:com.xxx.xxx:Launcher0
10-31 22:38:58.731 20733 20733 E Runtime : onTrimMemory level:80,pid:com.xxx.xxx:Launcher0
10-31 22:39:02.816 20733 20733 E Runtime : onTrimMemory level:80,pid:com.xxx.xxx:Launcher0

```

可以看出，在发生ANR的时间点前后，内存都处于紧张状态，level等级是80，查看Android API 文档；
可知80这个等级是很严重的，应用马上就要被杀死，被杀死的这个应用从名字可以看出来是桌面，连桌面都快要被杀死，那普通应用能好到哪里去呢？
一般来说，发生内存紧张，会导致多个应用发生ANR，所以在日志中如果发现有多个应用一起ANR了，可以初步判定，此ANR与你的应用无关。

## 6.系统服务超时导致ANR

系统服务超时一般会包含BinderProxy.transactNative关键字，请看如下日志：

```
"main" prio=5 tid=1 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x72bce768 self=0xb400e7ca23a0b010
  | sysTid=4647 nice=0 cgrp=default sched=1073741825/1 handle=0xe7cb4a3fb4f8
  | state=S schedstat=( 64276581 1072197042 239 ) utm=3 stm=3 core=0 HZ=100
  | stack=0xffffe912e000-0xffffe9130000 stackSize=8192KB
  | held mutexes=
  native: (backtrace::Unwind failed for thread 4647: Thread has not responded to signal in time)
  at libcore.io.Linux.access(Native method)
  at libcore.io.ForwardingOs.access(ForwardingOs.java:72)
  at libcore.io.BlockGuardOs.access(BlockGuardOs.java:73)
  at libcore.io.ForwardingOs.access(ForwardingOs.java:72)
  at android.app.ActivityThread$AndroidOs.access(ActivityThread.java:7584)
  at java.io.UnixFileSystem.checkAccess(UnixFileSystem.java:281)
  at java.io.File.exists(File.java:815)
  at android.app.ContextImpl.ensureExternalDirsExistOrFilter(ContextImpl.java:2892)
  at android.app.ContextImpl.getExternalFilesDirs(ContextImpl.java:766)
  - locked <0x07a54129> (a java.lang.Object)
  at android.content.ContextWrapper.getExternalFilesDirs(ContextWrapper.java:278)
  at k0.a$b.b(ContextCompat.java:-1)
  at k0.a.f(ContextCompat.java:-1)
  at androidx.core.content.FileProvider.parsePathStrategy(FileProvider.java:18)
  at androidx.core.content.FileProvider.getPathStrategy(FileProvider.java:3)
  - locked <0x09ed6cae> (a java.util.HashMap)
  at androidx.core.content.FileProvider.attachInfo(FileProvider.java:8)
  at android.app.ActivityThread.installProvider(ActivityThread.java:7287)
  at android.app.ActivityThread.installContentProviders(ActivityThread.java:6828)
  at android.app.ActivityThread.handleBindApplication(ActivityThread.java:6745)
  at android.app.ActivityThread.access$1300(ActivityThread.java:240)
  at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1920)
  at android.os.Handler.dispatchMessage(Handler.java:106)
  at android.os.Looper.loop(Looper.java:223)
  at android.app.ActivityThread.main(ActivityThread.java:7707)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:592)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:947)

```

以上可以看出来，是在从堆栈可以看出ANR：getExternalFilesDirs。

    at android.app.ActivityThread.installProvider(ActivityThread.java:7287)
    at android.app.ActivityThread.installContentProviders(ActivityThread.java:6828)
    at android.app.ActivityThread.handleBindApplication(ActivityThread.java:6745)

可以看出来是在AMS创建APP时出现的问题，因此可以给系统侧分析。

## 7.Input dispatching timed out

ANR部分日志信息：
PID: 32640
UID: 1110155
Input dispatching timed out (4e4035c com.aaa/com.aaa.home.MainActivity (server) is not responding. Waited 5001ms for MotionEvent(deviceId=-1, eventTime=85286596000000, source=0x00001002, displayId=0, action=DOWN, actionButton=0x00000000, flags=0x00000000, metaState=0x00000000, buttonState=0x00000000, classification=NONE, edgeFlags=0x00000000, xPrecision=1.0, yPrecision=1.0, xCursorPosition=nan, yCursorPosition=nan, pointers=[0: (525.0, 957.0)]), policyFlags=0x6b000000)

CPU usage from 0ms to 31081ms later (2024-10-22 07:03:11.876 to 2024-10-22 07:03:42.956):

这个导致的原因有很多，这个问题无法从ANR日志里去分析，就需要从系统日志里去查找问题，这个报错的时间上，需要往前去找日志信息，从日志里可以找到关键的信息，如下

10-22 07:03:07.795 17729 17830 I InputDispatcher: Not sending touch event to 4e4035c com.aaa/com.aaa.home.MainActivity because it is paused

点击事件没有发送成功，说明这里就是发生问题的关键时间点，顺藤摸瓜，往上找一下有没有什么错误信息，
```
10-22 07:03:07.634 1015 15113 D EMS:ExceptionHandlerFactory: Get handler : STRICT_MODE
10-22 07:03:07.634 1015 15113 E JavaBinder: *** Uncaught remote exception! (Exceptions are not yet supported across processes.)
10-22 07:03:07.634 1015 15113 E JavaBinder: java.util.concurrent.RejectedExecutionException: Task c.a.a.a.u@4f72b0e rejected from java.util.concurrent.ThreadPoolExecutor@735f423[Running, pool size = 8, active threads = 8, queued tasks = 6, completed tasks = 19620]
10-22 07:03:07.634 1015 15113 E JavaBinder: at java.util.concurrent.ThreadPoolExecutorAbortPolicy.rejectedExecution(ThreadPoolExecutor.java:2086)10−2207:03:07.634101515113EJavaBinder:atjava.util.concurrent.ThreadPoolExecutor.reject(ThreadPoolExecutor.java:848)10−2207:03:07.634101515113EJavaBinder:atjava.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1394)10−2207:03:07.634101515113EJavaBinder:atc.a.a.a.v.a(UnknownSource:2)10−2207:03:07.634101515113EJavaBinder:atc.a.a.a.t.p(UnknownSource:104)10−2207:03:07.634101515113EJavaBinder:atc.a.a.a.t.j(UnknownSource:102)10−2207:03:07.634101515113EJavaBinder:atc.b.a.bAbortPolicy.rejectedExecution(ThreadPoolExecutor.java:2086) 10-22 07:03:07.634 1015 15113 E JavaBinder: at java.util.concurrent.ThreadPoolExecutor.reject(ThreadPoolExecutor.java:848) 10-22 07:03:07.634 1015 15113 E JavaBinder: at java.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1394) 10-22 07:03:07.634 1015 15113 E JavaBinder: at c.a.a.a.v.a(Unknown Source:2) 10-22 07:03:07.634 1015 15113 E JavaBinder: at c.a.a.a.t.p(Unknown Source:104) 10-22 07:03:07.634 1015 15113 E JavaBinder: at c.a.a.a.t.j(Unknown Source:102) 10-22 07:03:07.634 1015 15113 E JavaBinder: at c.b.a.bAbortPolicy.rejectedExecution(ThreadPoolExecutor.java:2086)10−2207:03:07.634101515113EJavaBinder:atjava.util.concurrent.ThreadPoolExecutor.reject(ThreadPoolExecutor.java:848)10−2207:03:07.634101515113EJavaBinder:atjava.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1394)10−2207:03:07.634101515113EJavaBinder:atc.a.a.a.v.a(UnknownSource:2)10−2207:03:07.634101515113EJavaBinder:atc.a.a.a.t.p(UnknownSource:104)10−2207:03:07.634101515113EJavaBinder:atc.a.a.a.t.j(UnknownSource:102)10−2207:03:07.634101515113EJavaBinder:atc.b.a.ba.onTransact(Unknown Source:207)
10-22 07:03:07.634 1015 15113 E JavaBinder: at android.os.Binder.execTransactInternal(Binder.java:1179)
10-22 07:03:07.634 1015 15113 E JavaBinder: at android.os.Binder.execTransact(Binder.java:1143)
```

1015进程频繁过量使用binder线程导致系统Binder线程池满载, 带来后果就是系统做binder进程通讯变慢亦或停止，界面点击响应不及时

如下图: 1015进程频繁抛出RejectedExcutionException, 该进程频繁进行binder进程间通信，使得系统binder线程池满载(Runing 8个+ active 8个共计16个)，导致其他程序在和system_server进程间通信过程中存在等待延迟，拖慢点击事件响应而出现ANR。


# 四、具体分析

## 1.wifi界面加载失败

### 分析trace日志

anr_2025-07-14-15-41-20-128

```
"main" prio=5 tid=1 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x7451a1f0 self=0xa65e1000
  | sysTid=19259 nice=-10 cgrp=default sched=0/0 handle=0xaa8ec494
  | state=S schedstat=( 1503048351 754306716 3104 ) utm=126 stm=24 core=0 HZ=100
  | stack=0xbb74c000-0xbb74e000 stackSize=8MB
  | held mutexes=
  native: #00 pc 00053f2c  /system/lib/libc.so (__ioctl+8)
  native: #01 pc 00021c11  /system/lib/libc.so (ioctl+36)
  native: #02 pc 0003d5f7  /system/lib/libbinder.so (android::IPCThreadState::talkWithDriver(bool)+206)
  native: #03 pc 0003e003  /system/lib/libbinder.so (android::IPCThreadState::waitForResponse(android::Parcel*, int*)+26)
  native: #04 pc 0003725d  /system/lib/libbinder.so (android::BpBinder::transact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+36)
  native: #05 pc 000c4c77  /system/lib/libandroid_runtime.so (android_os_BinderProxy_transact(_JNIEnv*, _jobject*, int, _jobject*, _jobject*, int)+82)
  at android.os.BinderProxy.transactNative(Native method)
  at android.os.BinderProxy.transact(Binder.java:1127)
  at android.net.wifi.IWifiManager$Stub$Proxy.getConfiguredNetworks(IWifiManager.java:892)
  at android.net.wifi.WifiManager.getConfiguredNetworks(WifiManager.java:1019)
  at com.android.settingslib.wifi.WifiTracker.fetchScansAndConfigsAndUpdateAccessPoints(WifiTracker.java:522)
  at com.android.settingslib.wifi.WifiTracker.forceUpdate(WifiTracker.java:337)
  at com.android.settingslib.wifi.WifiTracker.onStart(WifiTracker.java:300)
  at com.android.settingslib.core.lifecycle.Lifecycle.onStart(Lifecycle.java:120)
  at com.android.settingslib.core.lifecycle.Lifecycle.access$100(Lifecycle.java:54)
  at com.android.settingslib.core.lifecycle.Lifecycle$LifecycleProxy.onLifecycleEvent(Lifecycle.java:218)
  at java.lang.reflect.Method.invoke(Native method)
  at android.arch.lifecycle.ClassesInfoCache$MethodReference.invokeCallback(ClassesInfoCache.java:221)
  at android.arch.lifecycle.ClassesInfoCache$CallbackInfo.invokeMethodsForEvent(ClassesInfoCache.java:193)
  at android.arch.lifecycle.ClassesInfoCache$CallbackInfo.invokeCallbacks(ClassesInfoCache.java:185)
  at android.arch.lifecycle.ReflectiveGenericLifecycleObserver.onStateChanged(ReflectiveGenericLifecycleObserver.java:36)
  at android.arch.lifecycle.LifecycleRegistry$ObserverWithState.dispatchEvent(LifecycleRegistry.java:355)
  at android.arch.lifecycle.LifecycleRegistry.forwardPass(LifecycleRegistry.java:293)
  at android.arch.lifecycle.LifecycleRegistry.sync(LifecycleRegistry.java:333)
  at android.arch.lifecycle.LifecycleRegistry.moveToState(LifecycleRegistry.java:138)
  at android.arch.lifecycle.LifecycleRegistry.handleLifecycleEvent(LifecycleRegistry.java:124)
  at com.android.settingslib.core.lifecycle.ObservablePreferenceFragment.onStart(ObservablePreferenceFragment.java:79)
  at com.android.settings.wifi.WifiSettings.onStart(WifiSettings.java:345)
  at android.app.Fragment.performStart(Fragment.java:2548)
  at android.app.FragmentManagerImpl.moveToState(FragmentManager.java:1334)
  at android.app.FragmentManagerImpl.moveFragmentToExpectedState(FragmentManager.java:1576)
  at android.app.FragmentManagerImpl.moveToState(FragmentManager.java:1637)
  at android.app.FragmentManagerImpl.dispatchMoveToState(FragmentManager.java:3046)
  at android.app.FragmentManagerImpl.dispatchStart(FragmentManager.java:3003)
  at android.app.FragmentController.dispatchStart(FragmentController.java:193)
  at android.app.Activity.performStart(Activity.java:7165)
  at android.app.ActivityThread.handleStartActivity(ActivityThread.java:2937)
  at android.app.servertransaction.TransactionExecutor.performLifecycleSequence(TransactionExecutor.java:180)
  at android.app.servertransaction.TransactionExecutor.cycleToPath(TransactionExecutor.java:165)
  at android.app.servertransaction.TransactionExecutor.executeLifecycleState(TransactionExecutor.java:142)
  at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:70)
  at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1808)
  at android.os.Handler.dispatchMessage(Handler.java:106)
  at android.os.Looper.loop(Looper.java:193)
  at android.app.ActivityThread.main(ActivityThread.java:6669)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:493)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:858)
```


从提供的 **ANR 主线程堆栈** 来看，`com.android.settings` 的 ANR 发生在 **WifiSettings 启动阶段**，直接原因是 **主线程在等待 Binder 调用返回时被阻塞**，导致输入事件无法及时处理。以下是详细分析：


**1. 关键堆栈分析**

**（1）阻塞点：Binder 调用卡死**

native: #00 pc 00053f2c  /system/lib/libc.so (__ioctl+8)
native: #01 pc 00021c11  /system/lib/libc.so (ioctl+36)
native: #02 pc 0003d5f7  /system/lib/libbinder.so (android::IPCThreadState::talkWithDriver(bool)+206)
native: #03 pc 0003e003  /system/lib/libbinder.so (android::IPCThreadState::waitForResponse(android::Parcel*, int*)+26)
at android.os.BinderProxy.transactNative(Native method)
at android.os.BinderProxy.transact(Binder.java:1127)
at android.net.wifi.IWifiManager$Stub$Proxy.getConfiguredNetworks(IWifiManager.java:892)
at android.net.wifi.WifiManager.getConfiguredNetworks(WifiManager.java:1019)

- **`getConfiguredNetworks()`** 通过 Binder 调用 `IWifiManager` 服务（运行在 `system_server` 进程）获取 Wi-Fi 配置列表。
    
- **主线程卡在 `transactNative`**，说明 **Binder 调用未返回**，可能是：
    
    - **`system_server` 进程繁忙**（如高 CPU 负载、死锁）。
        
    - **Wi-Fi 服务（`wpa_supplicant` 或 HAL 层）响应慢**。
        
    - **Binder 驱动或内核问题**（如锁竞争、IO 阻塞）。
        

**（2）调用链**

WifiSettings.onStart()
→ WifiTracker.forceUpdate()
→ WifiTracker.fetchScansAndConfigsAndUpdateAccessPoints()
→ WifiManager.getConfiguredNetworks()
→ BinderProxy.transact()  # 阻塞点

- 这是 **Wi-Fi 设置页面启动时的正常流程**，但 Binder 调用超时导致 ANR。
    

 **2. 结合 CPU 负载分析**

在之前的日志中，**`system_server` 占用 21% CPU**，而 **`com.yc.smartdriver` 占用 123% CPU**，说明：

1. **`system_server` 可能因高负载无法及时处理 Binder 请求**。
    
2. **第三方进程（如 `smartdriver`）抢占了 CPU 资源**，间接导致系统服务响应延迟。
    

 **3. 根本原因**

 **（1）直接原因**

- **主线程在 `getConfiguredNetworks()` 的 Binder 调用中被阻塞**，超过 5 秒未返回（触发 ANR）。
    
- **阻塞的可能根源**：

| 可能性                            | 说明                                    |
| ------------------------------ | ------------------------------------- |
| **`system_server` 进程高负载**      | 其他 Binder 调用占用了 `system_server` 的线程池。 |
| **Wi-Fi 服务（wpa_supplicant）卡死** | 底层 Wi-Fi 驱动或 HAL 层响应慢。                |
| **Binder 内核驱动阻塞**              | 如 Binder 缓冲区满、锁竞争。                    |


 **（2）间接原因**

- **`com.yc.smartdriver` 占用 123% CPU**，导致系统整体负载过高（Load: 12.14），加剧了 Binder 调用延迟。


 **结论**

此次 ANR 的直接原因是 **主线程在 Binder 调用 `getConfiguredNetworks()` 时被阻塞**，而根本原因可能是：

1. **`system_server` 进程高负载**（受 `com.yc.smartdriver` 影响）。
    
2. **Wi-Fi 服务响应慢**（需排查底层驱动/HAL）。
    

**建议优先优化 Wi-Fi 配置的异步加载，并限制第三方进程的 CPU 占用。**


### 分析system日志

```
9 D/NetworkPolicy( 3229): packageName:com.fj.smartkit
07-14 15:57:39.843 E/ActivityManager( 3229): ANR in com.android.settings (com.android.settings/.Settings$WifiSettingsActivity)
07-14 15:57:39.843 E/ActivityManager( 3229): PID: 5725
07-14 15:57:39.843 E/ActivityManager( 3229): Reason: Input dispatching timed out (Waiting because no window has focus but there is a focused application that may eventually add a window when it finishes starting up.)
07-14 15:57:39.843 E/ActivityManager( 3229): Load: 12.14 / 8.3 / 4.83
07-14 15:57:39.843 E/ActivityManager( 3229): CPU usage from 0ms to 13895ms later (2025-07-14 15:57:25.918 to 2025-07-14 15:57:39.813):
07-14 15:57:39.843 E/ActivityManager( 3229):   123% 3908/com.yc.smartdriver: 55% user + 68% kernel / faults: 1149841 minor 1 major
07-14 15:57:39.843 E/ActivityManager( 3229):   86% 5626/app_process: 79% user + 6.7% kernel / faults: 15 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   36% 3023/adbd: 4.1% user + 32% kernel / faults: 475093 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   29% 4030/com.fj.smartkit: 20% user + 9% kernel / faults: 11304 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   21% 3229/system_server: 16% user + 5.1% kernel / faults: 6593 minor 1 major
07-14 15:57:39.843 E/ActivityManager( 3229):   11% 3394/com.android.systemui: 10% user + 1.2% kernel / faults: 6330 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   11% 3377/com.google.android.inputmethod.latin: 10% user + 1.1% kernel / faults: 5645 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   10% 3067/media.codec: 7.8% user + 3% kernel / faults: 3718 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   7% 2897/logd: 2.6% user + 4.3% kernel / faults: 3 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   5.6% 2962/android.hardware.audio@2.0-service: 0.6% user + 5% kernel
07-14 15:57:39.843 E/ActivityManager( 3229):   4.5% 4188/logcat: 1.7% user + 2.8% kernel / faults: 5591 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   3.8% 3516/com.android.phone: 2.7% user + 1.1% kernel / faults: 1144 minor 1 major
07-14 15:57:39.843 E/ActivityManager( 3229):   3.6% 3202/logcat: 1.4% user + 2.2% kernel / faults: 5590 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   2.9% 3500/com.droidlogic: 1.8% user + 1% kernel / faults: 1235 minor 2 major
07-14 15:57:39.843 E/ActivityManager( 3229):   2.3% 2987/surfaceflinger: 1.1% user + 1.2% kernel / faults: 177 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   2.3% 320/kworker/u8:3: 0% user + 2.3% kernel
07-14 15:57:39.843 E/ActivityManager( 3229):   2.2% 5581/kworker/0:1: 0% user + 2.2% kernel
07-14 15:57:39.843 E/ActivityManager( 3229):   2.1% 6/kworker/u8:0: 0% user + 2.1% kernel
07-14 15:57:39.843 E/ActivityManager( 3229):   1.8% 5773/kworker/u8:2: 0% user + 1.8% kernel
07-14 15:57:39.843 E/ActivityManager( 3229):   1.4% 2960/systemcontrol: 0.2% user + 1.1% kernel / faults: 557 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   1% 3850/com.example.hasim: 0.8% user + 0.2% kernel / faults: 848 minor 1 major
07-14 15:57:39.843 E/ActivityManager( 3229):   0.7% 5725/com.android.settings: 0.5% user + 0.2% kernel / faults: 862 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   0.5% 3058/media.extractor: 0.2% user + 0.2% kernel / faults: 1043 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   0% 3859/com.android.se: 0% user + 0% kernel / faults: 746 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   0.5% 4187/logcat: 0.2% user + 0.3% kernel / faults: 1 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   0.5% 3293/rild: 0.2% user + 0.2% kernel / faults: 275 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   0% 3873/com.droidlogic.SubTitleService: 0% user + 0% kernel / faults: 199 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   0.4% 3820/ksdioirqd/sdio: 0% user + 0.4% kernel
07-14 15:57:39.843 E/ActivityManager( 3229):   0.3% 422/kworker/u8:4: 0% user + 0.3% kernel
07-14 15:57:39.843 E/ActivityManager( 3229):   0.3% 3061/statsd: 0.1% user + 0.2% kernel / faults: 514 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   0.2% 7/rcu_preempt: 0% user + 0.2% kernel
07-14 15:57:39.843 E/ActivityManager( 3229):   0.2% 1896/irq/69-meson-am: 0% user + 0.2% kernel
07-14 15:57:39.843 E/ActivityManager( 3229):   0.2% 2973/android.hardware.graphics.composer@2.2-service: 0% user + 0.2% kernel / faults: 126 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   0.2% 3176/irq/44-ff660000: 0.2% user + 0% kernel
07-14 15:57:39.843 E/ActivityManager( 3229):   0.2% 3814/RTW_RECV_THREAD: 0% user + 0.2% kernel
07-14 15:57:39.843 E/ActivityManager( 3229):   0.2% 1830/kthread_di: 0% user + 0.2% kernel
07-14 15:57:39.843 E/ActivityManager( 3229):   0.2% 3209/process-tracker: 0.1% user + 0% kernel / faults: 85 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   0.2% 3815/RTW_CMD_THREAD: 0% user + 0.2% kernel
07-14 15:57:39.843 E/ActivityManager( 3229):   0.1% 1//init: 0% user + 0.1% kernel / faults: 141 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   0.1% 2898/servicemanager: 0% user + 0% kernel / faults: 84 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   0.1% 2957/hdmicecd: 0% user + 0.1% kernel / faults: 20 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   0% 1894/irq/68-meson-am: 0% user + 0% kernel
07-14 15:57:39.843 E/ActivityManager( 3229):   0% 1938/mmcqd/0: 0% user + 0% kernel
07-14 15:57:39.843 E/ActivityManager( 3229):   0% 2954/netd: 0% user + 0% kernel / faults: 119 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   0% 2976/android.hardware.memtrack@1.0-service: 0% user + 0% kernel / faults: 54 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   0% 2978/android.hardware.power@1.0-service: 0% user + 0% kernel
07-14 15:57:39.843 E/ActivityManager( 3229):   0% 3037/cameraserver: 0% user + 0% kernel / faults: 66 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   0% 3050/drmserver: 0% user + 0% kernel / faults: 63 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   0% 3057/mediadrmserver: 0% user + 0% kernel / faults: 30 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   0% 3072/tombstoned: 0% user + 0% kernel / faults: 160 minor
07-14 15:57:39.843 E/ActivityManager( 3229):   0% 3227/aml_buf_toggle_: 0% user + 0% kernel
07-14 15:57:39.843 E/ActivityManager( 3229):   0% 3295/kworker/u9:2: 0% user + 0% kernel
07-14 15:57:39.843 E/ActivityManager( 3229):   0% 4189/logcat: 0% user + 0% kernel
07-14 15:57:39.843 E/ActivityManager( 3229):  +0% 5786/ping: 0% user + 0% kernel
07-14 15:57:39.843 E/ActivityManager( 3229): 97% TOTAL: 57% user + 39% kernel + 0% iowait
07-14 15:57:39.843 E/ActivityManager( 3229): CPU usage from 70ms to 864ms later (2025-07-14 15:57:25.988 to 2025-07-14 15:57:26.782):
07-14 15:57:39.843 E/ActivityManager( 3229):   106% 3908/com.yc.smartdriver: 48% user + 58% kernel / faults: 25159 minor
07-14 15:57:39.843 E/ActivityManager( 3229):     43% 4098/DefaultDispatch: 4.8% user + 38% kernel
07-14 15:57:39.843 E/ActivityManager( 3229):     38% 3946/HeapTaskDaemo
```

从提供的 **ANR 日志** 来看，**`com.android.settings`** 发生了 ANR（`Input dispatching timed out`），并且 **CPU 负载极高**（`97% TOTAL`），多个进程占用了大量 CPU 资源。以下是详细分析：

|   |   |   |   |
|---|---|---|---|
|**TOTAL**|CPU 总使用率|< 70% (单核)|≥ 80% 可能导致卡顿或 ANR|
|**user**|用户态占用（App、系统服务逻辑）|-|过高表示应用代码或服务负载大|
|**kernel**|内核态占用（系统调用、驱动、Binder 通信等）|-|过高可能因频繁 IPC 或锁竞争|
|**iowait**|CPU 等待 I/O 操作的时间（如磁盘/网络读写）|越低越好|> 10% 表示 I/O 瓶颈|

 **1. ANR 直接原因**

日志显示：

E/ActivityManager( 3229): ANR in com.android.settings (com.android.settings/.Settings$WifiSettingsActivity)
E/ActivityManager( 3229): Reason: Input dispatching timed out (Waiting because no window has focus but there is a focused application that may eventually add a window when it finishes starting up.)

- **ANR 类型**：`Input dispatching timed out`（输入事件分发超时）。
    
- **可能原因**：
    - **主线程阻塞**（如 CPU 占用高、死锁、IO 等待）。
    - **窗口未及时创建**（Activity 启动慢）。
    - **系统负载过高**（CPU 被其他进程抢占）。
        

**2. CPU 负载分析**

日志显示 **CPU 总占用率高达 97%**（`57% user + 39% kernel`），其中：

**（1）高 CPU 占用进程**

|进程|CPU 占用|可能影响|
|---|---|---|
|**`com.yc.smartdriver` (3908)**|**123%**（55% user + 68% kernel）|**严重占用 CPU，可能是主因**|
|`app_process` (5626)|86%|可能运行 JVM 任务|
|`adbd` (3023)|36%|ADB 调试进程，异常高|
|`com.fj.smartkit` (4030)|29%|第三方应用占用高|
|`system_server` (3229)|21%|系统核心进程，正常偏高|
|`com.android.phone` (3516)|3.8%|电话服务，正常|

 **（2）关键发现**

- **`com.yc.smartdriver` 占用 CPU 异常高（123%）**，远超正常范围（通常单进程 <50%）。
    
- **`com.fj.smartkit` 也占用了 29% CPU**，可能与 `smartdriver` 相关（同厂商？）。
    
- **`adbd` 占用 36%**，可能因频繁日志输出或异常调试导致。


**3. 为什么高 CPU 会导致 ANR？**

1. **主线程无法获得 CPU 时间片**
    
    - Android 主线程（UI 线程）需要及时处理输入事件（如触摸、按键）。
        
    - 如果 **CPU 被其他进程（如 `com.yc.smartdriver`）占满**，主线程会被阻塞，导致输入事件超时（ANR）。
        
2. **`com.android.settings` 自身 CPU 占用低（0.7%）**
    
    - 说明 **ANR 不是由于 `Settings` 自身代码问题**，而是 **系统资源被抢占**。
        
3. **`Load: 12.14 / 8.3 / 4.83`**
    
    - 系统负载（Load Average）远高于 CPU 核心数（如 4 核设备负载 >4 即过载）。
        
    - 表明 **系统严重过载**，进程排队等待 CPU。


**4. 根本原因**

**ANR 的直接原因是 `com.yc.smartdriver` 和 `com.fj.smartkit` 占用了过多 CPU 资源**，导致：

1. **`com.android.settings` 主线程无法及时响应输入事件**。
    
2. **系统负载过高（Load 12.14）**，进一步加剧资源竞争。


**5. 解决方案**

 **（1）优化高 CPU 进程**

- **排查 `com.yc.smartdriver`**：
- **限制 `com.fj.smartkit` 的 CPU 使用**：

**（2）降低系统负载**

- **关闭不必要的后台进程**（如 `adbd` 如果不需要调试）。
- **检查是否有内存泄漏**（`adb shell dumpsys meminfo`），避免频繁 GC 加剧 CPU 压力。

**（3）增强 ANR 监控**

- 使用 `systrace` 或 `Android Profiler` 分析 CPU 调度问题：

- python systrace.py -t 10 -o trace.html sched freq idle am wm

- 检查 `traces.txt` 确认 `com.android.settings` 主线程是否阻塞。

 **6. 结论**

**本次 ANR 的主要原因是 `com.yc.smartdriver` 和 `com.fj.smartkit` 占用了过多 CPU 资源**，导致系统过载，`com.android.settings` 无法及时响应输入事件。  
**解决方向**：

1. 优化或限制高 CPU 进程。
2. 监控系统负载，避免资源耗尽。

如果需要更精准定位，建议提供：

- **`/data/anr/traces.txt`**（查看 ANR 时的线程堆栈）。

- **`adb shell top -H -p <PID_OF_SMARTDRIVER>`**（查看 `smartdriver` 的线程级 CPU 占用）。


### 分析event日志

在event中搜索am_anr

```
07-14 15:48:30.413 I/am_anr  ( 3229): [0,3536,com.android.settings,952647237,Input dispatching timed out (Waiting because no window has focus but there is a focused application that may eventually add a window when it finishes starting up.)]
```

### 分析dmesg日志




## 2.top相关命令

```
adb shell top


|`-n 1`|只刷新一次后退出|`adb shell top -n 1`|
|`-d 2`|刷新间隔（秒）|`adb shell top -d 1`|
|`-m 10`|显示前 10 个进程|`adb shell top -m 10`|
|`-s cpu`|按 CPU 占用排序|`adb shell top -s cpu`|
|`-s rss`|按内存占用排序|`adb shell top -s rss`|
|`-t`|显示线程信息|`adb shell top -t`|
|`-H`|显示线程详情（结合 `-t`）|`adb shell top -H -t`|
```

User 10%, System 5%, IOW 0%, IRQ 0%
PID PR CPU% S  #THR     VSS     RSS PCY UID      Name
1234  0  15% S   45 123456K  78901K  fg u0_a123  com.example.app

- `PID`：进程 ID
    
- `PR`：优先级
    
- `CPU%`：CPU 占用百分比
    
- `S`：进程状态（`R`=运行, `S`=睡眠, `Z`=僵尸进程）
    
- `#THR`：线程数
    
- `VSS`：虚拟内存占用（KB）
    
- `RSS`：实际物理内存占用（KB）
    
- `PCY`：调度策略（`fg`=前台, `bg`=后台, `un`=未知）
    
- `UID`：用户 ID
    
- `Name`：进程名


### **(1) 监控某个应用的 CPU 和内存**

adb shell top -d 1 | grep "com.example.app"

输出：
1234  0  25% S   50 200MB   100MB  fg u0_a123  com.example.app

### **(2) 检查高 CPU 占用的进程**

adb shell top -n 1 -s cpu -m 5

输出：
PID USER     PR  NI CPU% VSS    RSS   PCY NAME
456 system   18  -2  30% 500MB 200MB  fg  system_server
789 u0_a123  10 -10  25% 300MB 150MB  fg  com.example.game

### **(3) 分析线程级别的 CPU 占用（适用于 ANR 排查）**

adb shell top -H -t -n 1 -s cpu | grep <PID>

先通过 `ps -A \| grep <包名>` 获取目标进程的 PID，再查看其线程。