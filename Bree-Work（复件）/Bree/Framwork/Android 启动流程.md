
[Android Framework系列（系统架构篇）_android framework 架构图-CSDN博客](https://blog.csdn.net/u013769274/article/details/118411915)


[Framework学习（三）之PMS、AMS、WMS_ams pms-CSDN博客](https://blog.csdn.net/ljx1400052550/article/details/115518631)

[Android系统启动-zygote篇 - Gityuan博客 | 袁辉辉的技术博客](https://gityuan.com/2016/02/13/android-zygote/)


[Android Framework 学习（二）：系统服务与应用服务 - 灰色飘零 - 博客园](https://www.cnblogs.com/renhui/p/12889077.html#:~:text=Android%20Framework%20%E5%AD%A6%E4%B9%A0%EF%BC%88%E4%BA%8C%EF%BC%89%EF%BC%9A%E7%B3%BB%E7%BB%9F%E6%9C%8D%E5%8A%A1%E4%B8%8E%E5%BA%94%E7%94%A8%E6%9C%8D%E5%8A%A1%201%201.%20%E5%90%AF%E5%8A%A8%E6%96%B9%E5%BC%8F%E7%9A%84%E5%8C%BA%E5%88%AB%20%E7%B3%BB%E7%BB%9F%E6%9C%8D%E5%8A%A1%E5%90%AF%E5%8A%A8%EF%BC%9A%E7%B3%BB%E7%BB%9F%E6%9C%8D%E5%8A%A1%E5%A4%A7%E9%83%A8%E5%88%86%E8%B7%91%E5%9C%A8system%20server%E9%87%8C%EF%BC%8C%E4%B8%80%E8%88%AC%E9%83%BD%E6%98%AF%E5%9C%A8system,%2F%2F%E9%80%9A%E8%BF%87context%E7%9A%84getSystemService%28%29%EF%BC%8C%E4%BC%A0%E5%85%A5%E5%90%8D%E5%AD%97%EF%BC%8C%E6%9F%A5%E5%88%B0%E6%9C%8D%E5%8A%A1%E7%9A%84%E7%AE%A1%E7%90%86%E5%AF%B9%E8%B1%A1%20PowerManager%20pm%20%20%3D%20%20context.getSystemService%28Context.POWER_SERVICE%29%3B%20)


[Framework学习（三）之PMS、AMS、WMS_ams pms-CSDN博客](https://blog.csdn.net/ljx1400052550/article/details/115518631)


[揭秘Android系统：全方位解析核心服务与功能列表 - 云原生实践](https://www.oryoy.com/news/jie-mi-android-xi-tong-quan-fang-wei-jie-xi-he-xin-fu-wu-yu-gong-neng-lie-biao.html)



[Android启动流程深度解析：从系统启动到应用启动全过程详解 - 云原生实践](https://www.oryoy.com/news/android-qi-dong-liu-cheng-shen-du-jie-xi-cong-xi-tong-qi-dong-dao-ying-yong-qi-dong-quan-guo-cheng-x.html)


# 1.启动流程概览

**Boot ROM → Bootloader → Linux Kernel → Init → Zygote → SystemServer → Launcher**

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
# 3. 关键分区与文件

|**分区**|内容|作用|
|---|---|---|
|`boot`|内核 + `ramdisk`|初始启动环境。|
|`system`|`/system` 镜像|Android 框架和系统应用。|
|`vendor`|厂商 HAL 驱动|硬件抽象层实现。|
|`userdata`|`/data` 用户数据|应用安装和配置。|
|`init.rc`|初始化脚本|定义服务启动顺序。|

# 4. 启动优化技巧

- **内核裁剪**：移除无用驱动，减少初始化时间。

- **并行启动**：在 `init.rc` 中优化服务启动顺序。

- **延迟加载**：非关键服务（如蓝牙）按需启动。
# 5. 调试方法

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
# 6. 启动时序图

![[启动时序图.png]]
# 7. 常见问题

#### **Q1：卡在开机动画**
- **排查**：
```
adb shell dmesg | grep "init"  # 检查内核日志
adb logcat | grep "ZYGOTE"    # 查看 Zygote 是否崩溃
```
#### **Q2：启动慢**
- **优化**：
	减少 `init.rc` 中的服务依赖。
	使用 `bootchart` 分析启动耗时：
```
adb shell 'touch /data/bootchart/enabled'
adb reboot
```

掌握 Android 启动流程有助于：
- 定制 ROM（如修改启动服务）。
- 解决开机故障。
- 优化系统性能。

