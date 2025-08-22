# 一、定义AIDL接口

AIDL:Android Interface Definition Language

aml_sdk/frameworks/base/core/java/android/os/IDhcpManager.aidl

```
package android.os;

interface IDhcpManager {
    // 设置 DHCP 范围
    void setDhcpRange(String iface, String startIp, String endIp, int leaseTime);
    
    // 获取当前配置（可选）
    String getDhcpRange(String iface);
}
```


# 二、实现Binder服务（添加Service)

aml_sdk/frameworks/base/services/core/java/com/android/server/DhcpManagerService.java

```
package com.android.server;

import android.os.IDhcpManager;
import android.util.Log;
import android.content.Context;

public class DhcpManagerService extends IDhcpManager.Stub {
    private static final String TAG = "DhcpManagerService";

    private final Context mContext;

    public DhcpManagerService(Context context) {
        mContext = context;
        Log.d(TAG, "DhcpManagerService");
    }

    @Override
    public void setDhcpRange(String iface, String startIp, String endIp, int leaseTime) {
        // 实现逻辑
    }

    @Override
    public String getDhcpRange(String iface) {
        // 实现逻辑
    }

}

```


# 三、添加编译规则Android.bp

aml_sdk/frameworks/base/Android.bp

```
java_library {
    name: "framework",

    srcs: [
		......
		"core/java/android/os/IDhcpManager.aidl",
		......
```


# 四、添加开机服务SystemServer

aml_sdk/frameworks/base/services/java/com/android/server/SystemServer.java

startOtherServices中添加

```
	DhcpManagerService dhcpService = null;

		traceBeginAndSlog("DhcpManagerService");
			try {
				dhcpService = new DhcpManagerService(context);
				ServiceManager.addService("dhcp", dhcpService);
			} catch (Throwable e) {
				reportWtf("starting dhcp service", e);
			}
		traceEnd();
```


# 五、注册服务SystemServiceRegistry(DHCP暂省略)

aml_sdk/frameworks/base/core/java/android/app/SystemServiceRegistry.java

在 Android 框架中添加系统服务时，是否需要向 `SystemServiceRegistry.java` 注册，取决于服务的调用方式：

|**服务类型**|**注册位置**|**调用方式**|适用场景|
|---|---|---|---|
|**核心系统服务**|`SystemServiceRegistry`|`Context.getSystemService()`|对应用暴露的公共服务|
|**内部系统服务**|仅注册到 `ServiceManager`|`ServiceManager.getService()`|系统内部组件间通信|

 **注册示例（非必要步骤）：**

```
// 在 SystemServiceRegistry.java 的 static {} 块中添加
registerService(
    Context.DHCP_SERVICE,  // 需先在 Context.java 中定义常量
    DhcpManagerService.class,
    serviceFetcher -> new DhcpManagerService()
);
```

```
registerService(Context.FANC_SERVICE, FancManager.class,
new CachedServiceFetcher<FancManager>() {
@Override
public FancManager createService(ContextImpl ctx) {
return new FancManager();
}});
```

 **验证服务是否注册成功**

```
adb shell service list | grep dhcp
```

预期输出：

dhcp: [android.os.IDhcpManager]

# 五、Context定义(DHCP暂省略)

frameworks\base\core\java\android\content\Context.java

```
public static final String FANC_SERVICE = "fanc";
```

# 六、Manager(DHCP暂省略)

frameworks\base\core\java\android\app\FancManager.java


# 七、添加Selinux权限(DHCP暂省略)

```
涉及文件：
system\sepolicy\prebuilts\api\29.0\public\service.te
● system\sepolicy\prebuilts\api\28.0\public\service.te
● system\sepolicy\prebuilts\api\27.0\public\service.te
● system\sepolicy\prebuilts\api\26.0\public\service.te
● system\sepolicy\public\service.te
●
添加以下修改：
type fanc_service, system_api_service, system_server_service, service_manager_type;

涉及文件：
system\sepolicy\prebuilts\api\29.0\private\service_contexts
● system\sepolicy\prebuilts\api\28.0\private\service_contexts
● system\sepolicy\prebuilts\api\27.0\private\service_contexts
● system\sepolicy\prebuilts\api\26.0\private\service_contexts

●
system\sepolicy\private\service_contexts

添加以下修改：
● fanc
u:object_r:fanc_service:s0
```

上述步骤修改完毕后优先编译selinux，确认权限添加正确：

```
mmm system/sepolicy -j36
```

整编后验证：

```
service list |grep fanc
```
# 八、app调用

更新jar包

```
aml_sdk/out/target/common/obj/JAVA_LIBRARIES/framework_intermediates/classes.jar
```

```
import android.os.IDhcpManager;  
import android.os.ServiceManager;

	try {
        IDhcpManager dhcpService = IDhcpManager.Stub.asInterface(
	        ServiceManager.getService("dhcp"));
		String str = dhcpService.getDhcpRange("eth0");
	} catch (Exception e) {
		throw new RuntimeException(e);
	}
```

 IDhcpManager dhcpService = IDhcpManager.Stub.asInterface(
        ServiceManager.getService("dhcp"));



