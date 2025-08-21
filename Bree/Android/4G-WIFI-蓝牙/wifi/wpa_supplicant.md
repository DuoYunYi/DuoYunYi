# Android P (9.0) 中 wpa_supplicant 启动流程详解

Android P 的 wpa_supplicant 启动流程相比之前版本有所优化，以下是完整的启动过程分析：

## 1. 启动触发入口

流程从 `WifiNative.startSupplicant()` 开始：

java

// frameworks/opt/net/wifi/service/java/com/android/server/wifi/WifiNative.java
public boolean startSupplicant() {
    synchronized (sLock) {
        return startSupplicantNative();
    }
}

## 2. JNI 层调用

通过 JNI 调用本地方法：

java

// frameworks/opt/net/wifi/service/jni/wifi.cpp
static jboolean android_net_wifi_startSupplicant(JNIEnv* env, jclass) {
    return ::wifi_start_supplicant() == 0;
}

## 3. HIDL 接口调用 (Android P 新特性)

cpp

// hardware/interfaces/wifi/supplicant/1.0/default/supplicant.cpp
Return<bool> Supplicant::start() {
    return startSupplicantInternal();
}

static int startSupplicantInternal() {
    property_set("ctl.start", "wpa_supplicant");
    // 等待启动完成
    for (int i = 0; i < 10; i++) {
        if (property_get("init.svc.wpa_supplicant", value, NULL) > 0) {
            if (strcmp(value, "running") == 0) return 0;
        }
        usleep(100000);
    }
    return -1;
}

## 4. init 进程处理

### 4.1 rc 文件定义 (Android P 新位置)

bash

# system/core/rootdir/etc/init.wifi.rc
service wpa_supplicant /vendor/bin/hw/wpa_supplicant \
    -O/data/vendor/wifi/sockets -e/data/vendor/wifi/entropy.bin \
    -g@android:wpa_wlan0
    class main
    socket wpa_wlan0 dgram 660 wifi wifi
    disabled
    oneshot

### 4.2 关键变化 (Android P)

- 可执行文件路径改为 `/vendor/bin/hw/wpa_supplicant`
    
- 数据目录改为 `/data/vendor/wifi/`
    
- 增加了 HIDL 兼容层
    

## 5. wpa_supplicant 主流程

### 5.1 新版初始化流程

cpp

// hardware/interfaces/wifi/supplicant/1.0/default/main.cpp
int main() {
    configureRpcThreadpool(1, true);
    sp<Supplicant> supplicant = new Supplicant();
    registerAsService();
    joinRpcThreadpool();
}

### 5.2 驱动加载改进

Android P 引入更强的驱动兼容性检查：

c

// external/wpa_supplicant_8/wpa_supplicant/drivers.c
if (wpa_drivers[i]->name == NULL || 
    strcmp(wpa_drivers[i]->name, "nl80211") != 0) {
    continue; // 只加载nl80211驱动
}

## 6. 启动后验证

### 6.1 新验证方法 (Android P)

bash

adb shell getprop init.svc.wpa_supplicant
# 应返回 "running"

adb shell dumpsys wifi | grep "Supplicant state"
# 应显示 "COMPLETED"

### 6.2 新增调试命令

bash

adb shell cmd wifi force-supplicant-start
adb shell cmd wifi force-supplicant-stop

## 7. 错误处理改进

### 7.1 新增错误码 (Android P)

|错误码|含义|
|---|---|
|0x0001|驱动加载失败|
|0x0002|配置解析失败|
|0x0003|HIDL接口绑定失败|

### 7.2 增强日志标记

bash

adb logcat -s WifiHAL SupplicantStaIfaceHal

## 8. 与之前版本的主要区别

1. **HIDL 化**：通过硬件抽象层接口控制
    
2. **vendor分区隔离**：可执行文件移到vendor分区
    
3. **强化的权限控制**：使用更严格的SELinux策略
    
4. **改进的启动超时处理**：从5秒缩短到1秒检测
    
5. **增强的驱动兼容性**：强制使用nl80211驱动
    

这个流程展示了Android P在WiFi架构上的重要改进，通过HIDL接口实现了更好的模块化和vendor隔离，同时保持了向后兼容性。