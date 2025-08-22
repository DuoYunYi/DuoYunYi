
# 编译

./makeImg-v8.sh chinayc


I/InputDispatcher( 3229): Application is not responding: AppWindowToken{c38f3ed token=Token{31f7204 ActivityRecord{26d4e17 u0 com.android.settings/.Settings$WifiSettingsActivity t178}}}.  It has been 5012.1ms since event, 5003.9ms since wait started.  Reason: Waiting because no window has focus but there is a focused application that may eventually add a window when it finishes starting up.
07-14 15:57:25.838 I/WindowManager( 3229): Input event dispatching timed out sending to application AppWindowToken{c38f3ed token=Token{31f7204 ActivityRecord{26d4e17 u0 com.android.settings/.Settings$WifiSettingsActivity t178}}}.  Reason: Waiting because no window has focus but there is a focused application that may eventually add a window when it finishes starting up.



从提供的日志信息来看，ANR（应用无响应）发生在`com.android.settings/.Settings$WifiSettingsActivity`，核心原因是**系统资源严重过载导致输入事件无法及时分发**，具体分析如下：


### 一、ANR 直接触发原因

日志明确记录 ANR 的触发原因：  
`Reason: Input dispatching timed out (Waiting because no window has focus but there is a focused application that may eventually add a window when it finishes starting up.)`

  

- 含义：输入事件（如用户点击、触摸）分发超时。系统处于 “无窗口获得焦点” 的状态，但存在一个 “已聚焦的应用”（可能是`WifiSettingsActivity`或其他应用），系统等待该应用完成启动并创建窗口，但等待超时。
- 本质：`WifiSettingsActivity`的主线程被阻塞，或因系统资源不足无法及时处理输入事件，导致用户操作无响应。

### 二、深层诱因：系统 CPU 资源严重过载

ANR 发生时的 CPU 使用数据（`CPU usage from 0ms to 13895ms later`）显示，系统处于**高负载、高占用**状态，是导致 ANR 的核心原因：

  

1. **总 CPU 使用率接近饱和**  
    日志显示`97% TOTAL: 57% user + 39% kernel`，说明 CPU 几乎被占满，用户态和内核态进程都在大量消耗资源，系统已无足够算力处理`WifiSettingsActivity`的输入事件。
    
2. **关键进程过度占用 CPU**  
    多个进程的 CPU 占用率异常偏高，直接挤压了`Settings`应用的资源：
    
    - `com.yc.smartdriver`（PID 3908）：占用 123% CPU（55% 用户态 + 68% 内核态），是资源消耗的 “重灾区”，可能在进行频繁的设备交互（如硬件通信、数据处理）。
    - `app_process`（PID 5626）：占用 86% CPU，该进程通常与应用启动、系统服务相关，高占用可能意味着大量应用在频繁启动或系统服务异常。
    - `adbd`（PID 3023）：占用 36% CPU，`adbd`是 Android 调试桥进程，高占用可能因频繁的调试操作（如日志输出、文件传输）导致。
3. **系统负载过高**  
    日志中`Load: 12.14 / 8.3 / 4.83`表示系统 1 分钟、5 分钟、15 分钟的平均负载，远高于正常水平（正常负载应接近 CPU 核心数，若为 4 核 CPU，负载 > 4 即视为过载）。高负载意味着系统中大量进程在排队等待 CPU 资源，`WifiSettingsActivity`的主线程也被 “插队”，无法及时响应。
    

### 三、其他潜在影响因素

1. **SELinux 权限警告（非直接原因，但可能间接影响）**  
    日志开头的`avc: denied { getattr } for path="/dev/ttyS2"`表示`platform_app`进程尝试访问`/dev/ttyS2`（串口设备）时被 SELinux 拒绝（但`permissive=1`表示宽容模式，仅记录不阻断）。若`com.yc.smartdriver`等进程依赖该设备通信，可能因权限问题导致异常重试，进一步消耗 CPU 资源。
    
2. **`WifiSettingsActivity`自身可能的主线程阻塞**  
    `WifiSettingsActivity`负责处理 WiFi 相关配置（如扫描热点、加载网络列表），若其主线程执行了耗时操作（如同步网络请求、大量 IO 操作），在系统资源本就紧张的情况下，会加剧 ANR 的触发。
    

### 总结

ANR 的核心原因是 **`com.yc.smartdriver`、`app_process`、`adbd`等进程过度占用 CPU，导致系统负载过高 **，使得`WifiSettingsActivity`的主线程无法及时处理输入事件，最终触发输入分发超时。解决需优先排查高 CPU 占用进程的异常行为（如优化`com.yc.smartdriver`的数据处理逻辑、减少`adbd`的调试负载），同时避免`WifiSettingsActivity`主线程执行耗时操作。




![[img_v3_02o7_59615112-6fb6-4006-a149-44d08c626c7g.jpg]]


![[Pasted image 20250715203433.png]]



07-14 15:57:03.166 V/StatusBar( 3394): mStatusBarWindow: com.android.systemui.statusbar.phone.StatusBarWindowView{e242469 V.ED..... ........ 0,0-1280,800} canPanelBeCollapsed(): true
07-14 15:57:03.180 E/HWComposer( 2987): getLayerReleaseFence failed for display -1: Invalid display
07-14 15:56:54.678 D/NetworkPolicy( 3229): packageName:com.fj.smartkit
07-14 15:57:03.184 I/ActivityManager( 3229): START u0 {act=android.settings.WIFI_SETTINGS flg=0x14000000 cmp=com.android.settings/.Settings$WifiSettingsActivity} from uid 10015
07-14 15:57:03.180 I/chatty  ( 2987): uid=1000(system) /system/bin/surfaceflinger identical 6 lines
07-14 15:57:03.180 E/HWComposer( 2987): getLayerReleaseFence failed for display -1: Invalid display


7-14 15:57:25.835 I/InputDispatcher( 3229): Application is not responding: AppWindowToken{c38f3ed token=Token{31f7204 ActivityRecord{26d4e17 u0 com.android.settings/.Settings$WifiSettingsActivity t178}}}.  It has been 5012.1ms since event, 5003.9ms since wait started.  Reason: Waiting because no window has focus but there is a focused application that may eventually add a window when it finishes starting up.
07-14 15:57:25.838 I/WindowManager( 3229): Input event dispatching timed out sending to application AppWindowToken{c38f3ed token=Token{31f7204 ActivityRecord{26d4e17 u0 com.android.settings/.Settings$WifiSettingsActivity t178}}}.  Reason: Waiting because no window has focus but there is a focused application that may eventually add a window when it finishes starting up.



# 其他分析

这个anr是跟wifi服务通信的时候阻塞了

wifi服务估计出问题了

同时要把wifi配置获取改成异步操作，这个操作在主线程导致主线程阻塞，发生了anr

ANR发生在Binder IPC调用中，具体在： 客户端: Settings应用主线程 服务端: system_server中的WiFi服务 接口: `IWifiManager.getConfiguredNetworks()`: 服务端处理超时，导致客户端无限等待


原代码

```
public List<WifiConfiguration> getConfiguredNetworks() {
        try {
            ParceledListSlice<WifiConfiguration> parceledList =
                mService.getConfiguredNetworks();
            if (parceledList == null) {
                return Collections.emptyList();
            }
            return parceledList.getList();
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```



修改：## **使用 `CompletableFuture`**

```
public CompletableFuture<List<WifiConfiguration>> getConfiguredNetworksAsync() {
    return CompletableFuture.supplyAsync(() -> {
        try {
            ParceledListSlice<WifiConfiguration> parceledList = mService.getConfiguredNetworks();
            return parceledList != null ? parceledList.getList() : Collections.emptyList();
        } catch (RemoteException e) {
            throw new RuntimeException(e); // 或自定义异常
        }
    }, Executors.newCachedThreadPool()); // 可替换为自定义线程池
}
```


调用方式
```
getConfiguredNetworksAsync()
    .thenAccept(result -> Log.d("Wifi", "Networks: " + result))
    .exceptionally(e -> { Log.e("Wifi", "Error", e); return null; });
```



简单用法：省略线程池，使用默认 commonPool
```
public CompletableFuture<List<WifiConfiguration>> getConfiguredNetworksAsync() {
    return CompletableFuture.supplyAsync(() -> {
        try {
            ParceledListSlice<WifiConfiguration> parceledList = mService.getConfiguredNetworks();
            return parceledList != null ? parceledList.getList() : Collections.emptyList();
        } catch (RemoteException e) {
            throw new RuntimeException("Failed to get WiFi configs", e);
        }
    }); // 省略线程池，使用默认 commonPool
}
```


```
public CompletableFuture<List<WifiConfiguration>> getConfiguredNetworksAsync() {
    return CompletableFuture.supplyAsync(() -> {
        try {
            ParceledListSlice<WifiConfiguration> parceledList = mService.getConfiguredNetworks();
            return parceledList != null ? parceledList.getList() : Collections.emptyList();
        } catch (RemoteException e) {
            throw new RuntimeException(e); // 或自定义异常
        }
    }); 
}
```

# 相关知识

# CompletableFuture




# 同步和异步
