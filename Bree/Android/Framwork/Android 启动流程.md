
[Android Framework系列（系统架构篇）_android framework 架构图-CSDN博客](https://blog.csdn.net/u013769274/article/details/118411915)

[Android Framework 学习（二）：系统服务与应用服务 - 灰色飘零 - 博客园](https://www.cnblogs.com/renhui/p/12889077.html#:~:text=Android%20Framework%20%E5%AD%A6%E4%B9%A0%EF%BC%88%E4%BA%8C%EF%BC%89%EF%BC%9A%E7%B3%BB%E7%BB%9F%E6%9C%8D%E5%8A%A1%E4%B8%8E%E5%BA%94%E7%94%A8%E6%9C%8D%E5%8A%A1%201%201.%20%E5%90%AF%E5%8A%A8%E6%96%B9%E5%BC%8F%E7%9A%84%E5%8C%BA%E5%88%AB%20%E7%B3%BB%E7%BB%9F%E6%9C%8D%E5%8A%A1%E5%90%AF%E5%8A%A8%EF%BC%9A%E7%B3%BB%E7%BB%9F%E6%9C%8D%E5%8A%A1%E5%A4%A7%E9%83%A8%E5%88%86%E8%B7%91%E5%9C%A8system%20server%E9%87%8C%EF%BC%8C%E4%B8%80%E8%88%AC%E9%83%BD%E6%98%AF%E5%9C%A8system,%2F%2F%E9%80%9A%E8%BF%87context%E7%9A%84getSystemService%28%29%EF%BC%8C%E4%BC%A0%E5%85%A5%E5%90%8D%E5%AD%97%EF%BC%8C%E6%9F%A5%E5%88%B0%E6%9C%8D%E5%8A%A1%E7%9A%84%E7%AE%A1%E7%90%86%E5%AF%B9%E8%B1%A1%20PowerManager%20pm%20%20%3D%20%20context.getSystemService%28Context.POWER_SERVICE%29%3B%20)


[Framework学习（三）之PMS、AMS、WMS_ams pms-CSDN博客](https://blog.csdn.net/ljx1400052550/article/details/115518631)


[揭秘Android系统：全方位解析核心服务与功能列表 - 云原生实践](https://www.oryoy.com/news/jie-mi-android-xi-tong-quan-fang-wei-jie-xi-he-xin-fu-wu-yu-gong-neng-lie-biao.html)



[Android启动流程深度解析：从系统启动到应用启动全过程详解 - 云原生实践](https://www.oryoy.com/news/android-qi-dong-liu-cheng-shen-du-jie-xi-cong-xi-tong-qi-dong-dao-ying-yong-qi-dong-quan-guo-cheng-x.html)


# 1.启动流程概览

Boot ROM → Bootloader → Linux Kernel → Init → Zygote → SystemServer → Launcher

# 2.详细阶段解析

## （1）BootROM(芯片厂商固件)

**作用**：CPU 上电后执行固化在芯片中的代码，初始化基础硬件（时钟、内存等）。

**关键动作**：加载 Bootloader 到内存（从 `bootloader` 分区）。
## **（2）Bootloader(引导加载程序)**

**常见 Bootloader**：高通：`Little Kernel (LK)`      联发科：`U-Boot`

**任务**：
	1.硬件自检（RAM、存储等）。    
	2.加载内核和 `initramfs`（从 `boot` 分区）。
	3.验证系统签名（安全启动）。
	4.跳转到内核入口点。
## **（3）Linux 内核启动**

**关键步骤**：
    解压内核（如 `zImage`）。
    初始化调度、内存管理、驱动（通过设备树 `DTB`）。
    挂载临时根文件系统（`ramdisk`）。
    启动第一个用户空间进程 **`init`**（PID=1）。
## **（4）Init 进程**

**阶段1**（`ramdisk` 中的 `/init`）：
    创建设备节点（`/dev`）。
    挂载 `/proc`、`/sys` 等虚拟文件系统。
    解析 `init.rc` 脚本（Android Init Language）。
**阶段2**：
    - 挂载 `/system`、`/vendor`、`/data` 分区。
    - 启动关键守护进程（`ueventd`、`logd`）。
## **（5）Zygote**

**作用**：预加载 Java 核心类（如 `ActivityThread`），孵化应用进
**启动流程**：
    1. `init` 解析 `zygote.rc`，启动 `app_process`。
    2. 预加载类和资源（通过 `DexOpt`）。
    3. 监听 Socket 等待孵化请求。
## **（6）SystemServer**

**核心服务**：
    `ActivityManagerService`（AMS）：管理应用生命周期。
    `PackageManagerService`（PMS）：应用安装与权限。
    `WindowManagerService`（WMS）：窗口管理。
        
**依赖顺序**：PowerManager → DisplayManager → AMS → PMS → WMS → Launcher
## **（7）Launcher**

**AMS** 启动默认桌面（如 `com.android.launcher3`）。
加载应用图标，响应用户交互。

![[Pasted image 20250724203358.png]]
# 3. 关键分区与文件

|**分区**|内容|作用|
|---|---|---|
|`boot`|内核 + `ramdisk`|初始启动环境。|
|`system`|`/system` 镜像|Android 框架和系统应用。|
|`vendor`|厂商 HAL 驱动|硬件抽象层实现。|
|`userdata`|`/data` 用户数据|应用安装和配置。|
|`init.rc`|初始化脚本|定义服务启动顺序。|

# 4. 调试方法

- **查看启动日志**：
```
adb logcat -b all | grep "boot"
```
- **测量启动时间**：
```
adb shell su root cat /proc/bootprof
```
- **定制启动动画**：  
```
替换 `/system/media/bootanimation.zip`。
```
# 5. 启动时序图

![[启动时序图.png]]
# 6.其他相关知识

`init进程是所有用户进程的鼻祖`

init进程还启动`servicemanager`(binder服务管家)、`bootanim`(开机动画)等重要服务

init进程孵化出Zygote进程，Zygote进程是Android系统的第一个Java进程(即虚拟机进程)，`Zygote是所有Java进程的父进程`，Zygote进程本身是由init进程孵化而来的。

Zygote进程，是由init进程通过解析init.rc文件后fork生成的

System Server进程，是由Zygote进程fork而来，`System Server是Zygote孵化的第一个进程`

Zygote进程孵化出的第一个App进程是Launcher



Linux现有管道、消息队列、共享内存、套接字、信号量、信号这些IPC机制，Android额外还有Binder IPC机制，Android OS中的Zygote进程的IPC采用的是Socket机制

## SystemServer，ServiceManager，SystemServiceManager的关系

在SystemServer进程中创建SystemServiceManager, ServiceManager是系统服务管理者,SysytemServiceManager启动一些继承自SystemService的服务，并将这些服务的Binder注册到ServiceManager中，对于其他的一些继承于IBinder的服务,通过ServiceMaanager的addService方法添加

**SystemServer：**
SystemServer是一个由zygote孵化出来的进程， 名字为system_server 。 SystemServer叫做系统服务进程，大部分Android提供的一些系统服务都运行在该进程中,包括AMS，WMS，PMS，这些系统的服务都是以一个线程的方式存在在SysyemServer进程中。

**SystemServiceManager：**
管理一些系统的服务，在SystemServer中初始化。启动各种系统服务：WMS/PMS/AMS等,调用ServerManager的addService方，将这些Service服务注册到ServerManager里面

**ServiceManager：**
ServiceManager像是一个路由，Service把自己注册在ServiceManager中,客户端 通过ServiceManager查询服务

1、维护一个svclist列表来存储service信息。
2、向客户端提供Service的代理，也就是BinderProxy。
3、维护一个死循环，不断的查看是否有service的操作请求，如果有就读取相应的内核binder driver。

## 孵化应用进程这种事为什么不交给SystemServer来做，而专门设计一个Zygote
- Zygote进程是所有Android进程的母体，包括system_server和各个App进程。zygote利用fork()方法生成新进程，对于新进程A复用Zygote进程本身的资源，再加上新进程A相关的资源，构成新的应用进程A。应用在启动的时候需要做很多准备工作，包括启动虚拟机，加载各类系统资源等等，这些都是非常耗时的，如果能在zygote里就给这些必要的初始化工作做好，子进程在fork的时候就能直接共享，那么这样的话效率就会非常高
- SystemServer里跑了一堆系统服务，这些不能继承到应用进程

## Zygote的IPC通信机制为什么使用socket而不采用binder
- Zygote是通过fork生成进程的
- 因为fork只能拷贝当前线程，不支持多线程的fork，fork的原理是copy-on-write机制，当父子进程任一方修改内存数据时（这是on-write时机），才发生缺页中断，从而分配新的物理内存（这是copy操作）。zygote进程中已经启动了虚拟机、进行资源和类的预加载以及各种初始化操作，App进程用时拷贝即可。Zygotefork出来的进程A只有一个线程，如果Zygote有多个线程，那么A会丢失其他线程。这时可能造成死锁。
- Binder通信需要使用Binder线程池,binder维护了一个16个线程的线程池，fork()出的App进程的binder通讯没法用


## 如何判断一个进程是有Zygote孵化还是System_Server

| **特征**         | **Zygote直接孵化**       | **system_server管理**           |
| -------------- | -------------------- | ----------------------------- |
| **父进程（PPID）**  | `zygote`/`zygote64`  | 可能是`system_server`            |
| **进程类型**       | Java应用/部分服务          | Native服务/部分系统组件               |
| **启动方式**       | `fork()` + Java类     | `Process.start()` 或 `init.rc` |
| **`cmdline`**  | Java类名（如`com.xxx`）   | 二进制路径（如`/system/bin/hw/xxx`）  |
| **SELinux上下文** | `u:r:app_process:s0` | 独立上下文（如`hal_xxx`）             |
**典型例子：**
1. **Zygote直接孵化**：
    - 普通APP（`com.tencent.mm`）
    - `systemui`（`com.android.systemui`）
    - `webview_zygote`的子进程
        
2. **system_server管理（但可能仍由Zygote fork）**：
    - `android.hardware.camera.provider@2.4-service`
    - `media.codec`（mediaserver相关）
        
3. **非Zygote孵化（直接由init启动）**：
    - `surfaceflinger`
    - `servicemanager`
    - `logd`