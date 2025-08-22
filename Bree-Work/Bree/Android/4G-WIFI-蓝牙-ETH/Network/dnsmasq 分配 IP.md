
1.setprop persist.后Dnsmasq服务启动

确认wlan和eth0的功能

桥接和dhcp分配的区别

# 一、Dnsmasq服务

## 1.使用命令开启

```
/system/bin/dnsmasq --keep-in-foreground --no-resolv --no-poll --dhcp-range=192.168.1.100,192.168.1.200,24h --pid-file=/data/dnsmasq.pid
```
## 2.在init.rc文件中配置dnsmasq

service dnsmasq /system/bin/dnsmasq \  # 定义服务名和可执行文件路径
    --keep-in-foreground \             # 保持前台运行（调试用）
    --no-resolv \                      # 不读取 /etc/resolv.conf
    --no-poll \                        # 不轮询 resolv.conf 变化
    --no-hosts \                       # 不读取 /etc/hosts
    --dhcp-range=${persist.dhcp.range.eth0} \  # DHCP IP 范围（需确保变量已定义）
    --interface=eth0 \                 # 只监听 eth0 接口
    --pid-file=/data/local/tmp/dnsmasq.pid  # PID 文件路径
    class main \                       # 服务类（main 表示主系统服务）
    user root \                        # 以 root 身份运行
    # disabled \                       # 注释掉表示不禁用
    # oneshot \                        # 注释掉表示非一次性服务
    seclabel u:r:dnsmasq:s0            # SELinux 安全上下文

```
// Bree for startup
service dnsmasq /system/bin/dnsmasq \
    --keep-in-foreground \
    --no-resolv \
    --no-poll \
    --no-hosts \
    --dhcp-range=${persist.dhcp.range.eth0} \
    --interface=eth0
    --pid-file=/data/local/tmp/dnsmasq.pid
    class main
    user root
    disabled
    # oneshot
    seclabel u:r:dnsmasq:s0

# 在 init.rc 中添加触发器
on property:persist.dhcp.start=1
    start dnsmasq

on property:persist.dhcp.stop=1
    stop dnsmasq

on property:persist.dhcp.range.*=*
    restart dnsmasq
```

setprop net.dnsmasq.start 1  # 启动服务
setprop net.dnsmasq.stop 1   # 停止服务

## 3.配置相关参数

| 参数                                             | 作用                                                           | Android 注意事项                         |
| ---------------------------------------------- | ------------------------------------------------------------ | ------------------------------------ |
| `--keep-in-foreground`                         | 阻止 `dnsmasq` 后台化，保持在前台运行                                     | **必须保留**，否则 Android 可能杀死进程           |
| `--no-resolv`                                  | 禁用自动读取 `/etc/resolv.conf`                                    | Android 无此文件，需禁用                     |
| `--no-poll`                                    | 禁用轮询 `/etc/resolv.conf` 的更改                                  | 避免无效操作，节省资源                          |
| `--no-hosts`                                   | 阻止 dnsmasq 从本地的 `/etc/hosts` 文件读取并加载 DNS 记录                  | 完全忽略 `/etc/hosts` 文件                 |
| `--dhcp-range=192.168.1.100,192.168.1.200,24h` | 定义 DHCP 分配的 IP 范围和租期                                         | 需匹配实际网络配置（如 `wlan0` 的子网）             |
| `--interface=eth0`                             | 明确指示 dnsmasq 只监听（listen on）名为 "eth0" 的网络接口，并只为该接口上的客户端请求提供服务 | **限定服务范围**：只监听指定接口，只为该网络内的客户端服务      |
| `--pid-file=/data/dnsmasq.pid`                 | 指定 PID 文件路径                                                  | **必须可写**，建议用 `/data/` 而非 `/var/run/` |

- **`class main`**  
    将服务归类到 `main` 类，与 Android 的 `init` 进程管理相关。
    - _用途_：`class_start main` 可启动所有属于 `main` 类的服务。
- **`user root`**  
    以 `root` 用户身份运行 `dnsmasq`。
    - _用途_：DHCP 和 DNS 服务通常需要较高权限（如绑定到 53/67 端口）。
- **`# disabled`**  
    注释行，表示服务默认不随类启动（需手动或条件触发）。
    - _用途_：避免开机自动运行，需显式调用 `start dnsmasq`。
- **`# oneshot`**  
    注释行，若启用则表示服务退出后不重启（此处未生效）。
    - _用途_：适用于一次性任务，但 DHCP/DNS 服务通常需持续运行。
- **`seclabel u:r:dnsmasq:s0`**  
    设置 SELinux 安全上下文，指定 `dnsmasq` 以 `dnsmasq` 域运行。
    - _用途_：满足 Android SELinux 策略，限制服务权限。

## 4.配置 DHCP 范围

在 `--dhcp-range` 中设置合理的 IP 范围，需与 Android 设备的接口 IP **同子网**。例如：
- 如果 `eth0` 的 IP 是 `192.168.1.2/24`，则 DHCP 范围可以是：

--dhcp-range=192.168.1.100,192.168.1.200,24h

避免与网关（如 `192.168.1.1`）或静态设备冲突。

## 5.允许 IP 转发（如需跨子网）

如果 Android 设备作为路由器，需启用 IP 转发：

echo 1 > /proc/sys/net/ipv4/ip_forward

并在 `init.rc` 中持久化：

on boot
    write /proc/sys/net/ipv4/ip_forward 1

## 6.其他命令

### （1）查看 dnsmasq 服务

```
ps -A | grep -i dnsmasq
```

### （2）检查 `dnsmasq` 是否绑定到正确接口

```
netstat -tuln | grep dnsmasq
```

验证监听的接口：netstat -tuln | grep -E '53|67'
正常应看到 `0.0.0.0:67`（DHCP）和 `0.0.0.0:53`（DNS）

### （3）确认 DHCP 请求到达 Android 设备

```
 tcpdump -i eth0 port 67 -n
```

### （4）查看 `dnsmasq` 分配的租约

```
cat /data/dnsmasq.leases  # 默认租约文件路径
```

### （5）android设备设置IP

使用ip命令（需要root）
```
su
ip addr add <IP地址>/<掩码位数> dev eth0
ip link set eth0 up
```

**示例**：
```
ip addr add 192.168.1.100/24 dev eth0
ip link set eth0 up
```

### （6）修改 pid 默认路径

aml_sdk/external/dnsmasq/src/config.h

```
#define RUNFILE "/data/local/tmp/dnsmasq.pid"
```

### （7）检查Android是否支持dnsmasq

```
adb shell ls /system/bin/dnsmasq
```

# 二、使用接口设置 eth0 IP

## 1.setConfiguration接口设置

```
public void setEthernetAddr(String iface, String ipAddr, int prefixLength) {
        try{
            // 1. 创建 StaticIpConfiguration 对象并配置
            StaticIpConfiguration staticIpConfig = new StaticIpConfiguration();
            // 设置IP地址和子网掩码 (e.g., 192.168.1.100/24)
            staticIpConfig.ipAddress = new LinkAddress(InetAddress.getByName(ipAddr), prefixLength);

            // 同样替换其他行：
            staticIpConfig.gateway = InetAddress.getByName(ipAddr);
            staticIpConfig.dnsServers.add(InetAddress.getByName("8.8.8.8"));
            staticIpConfig.dnsServers.add(InetAddress.getByName("8.8.4.4"));

            // 2. 创建 IpConfiguration 对象 (Android 9方式)
            IpConfiguration ipConfig = new IpConfiguration();
            // 使用setter方法进行配置
            ipConfig.setStaticIpConfiguration(staticIpConfig);
            // 如果需要DHCP，则使用：
            // ipConfig.setIpAssignment(IpAssignment.DHCP);
            
            ipConfig.setIpAssignment(IpAssignment.STATIC); 
            ipConfig.setProxySettings(IpConfiguration.ProxySettings.NONE); // 设置为无代理

            // 额外设置 IP 分配方式（关键修复！）
            // 使用反射设置 ipAssignment，因为某些版本没有公开的setter
            // Field ipAssignmentField = IpConfiguration.class.getDeclaredField("ipAssignment");
            // ipAssignmentField.setAccessible(true);
            // ipAssignmentField.set(ipConfig, IpAssignment.STATIC);
            // Log.d(TAG, "Set ipAssignment to STATIC via reflection");

            if (ethernetService != null) {
                //ethernetService.setConfiguration("eth0", ipConfig);
                ethernetService.setConfiguration(iface, ipConfig);
                Log.d(TAG, "Successfully set static IP configuration for eth0");
            } else {
                Log.d(TAG, "ethernetService is null");
            }
        } catch (UnknownHostException e) {
            e.printStackTrace();
        } catch (SecurityException e) {
            Log.e(TAG, "SecurityException: " + e.getMessage(), e);
        } catch (RemoteException e) {
            Log.e(TAG, "RemoteException: " + e.getMessage(), e);
        } catch (Exception e) {
            Log.e(TAG, "Unexpected exception: " + e.getMessage(), e);
        }
    }
```

ethernetService.setConfiguration(iface, ipConfig);单独使用接口设置时设置IP失败，
接口设置时提示：**EthernetTracker: updateIpConfiguration, iface: eth0, cfg: IP assignment: UNASSIGNED**

需要将IP assignment设置成static：**ipConfig.setIpAssignment(IpAssignment.STATIC)**; 

若ipConfig.setIpAssignment(IpAssignment.STATIC); 接口无法使用，可以使用反射调用：
```
            // 额外设置 IP 分配方式（关键修复！）
            // 使用反射设置 ipAssignment，因为某些版本没有公开的setter
            // Field ipAssignmentField = IpConfiguration.class.getDeclaredField("ipAssignment");
            // ipAssignmentField.setAccessible(true);
            // ipAssignmentField.set(ipConfig, IpAssignment.STATIC);
            // Log.d(TAG, "Set ipAssignment to STATIC via reflection");
```

## 2.app权限问题（先确定是否有权限问题）

aml_sdk/frameworks/opt/net/ethernet/java/com/android/server/ethernet/EthernetServiceImpl.java
```
/**
     * Set Ethernet configuration
     */
    @Override
    public void setConfiguration(String iface, IpConfiguration config) {
        if (!mStarted.get()) {
            Log.w(TAG, "System isn't ready enough to change ethernet configuration");
        }

        enforceConnectivityInternalPermission();

        if (mTracker.isRestrictedInterface(iface)) {
            enforceUseRestrictedNetworksPermission();
        }

        // TODO: this does not check proxy settings, gateways, etc.
        // Fix this by making IpConfiguration a complete representation of static configuration.
        mTracker.updateIpConfiguration(iface, new IpConfiguration(config));
    }
```

**enforceConnectivityInternalPermission** 对权限进行检查，
需要**android.permission.CONNECTIVITY_INTERNAL** 权限，但是普通应用无法获得

调试：将enforceConnectivityInternalPermission注释掉后功能正常
正式：需要将app设置为系统应用
# 三、Dnsmasq与wifi热点共享区别

- **dnsmasq (独立 DHCP 模式)**
    - 是一个轻量级的 DHCP 和 DNS 服务器工具，专注于提供 **IP 分配（DHCP）** 和 **域名解析（DNS）** 功能。
    - 通常用于小型网络（如家庭局域网、开发环境），配置灵活，可自定义 IP 范围、租期、静态绑定等。
    - 仅负责分配 IP，不涉及网络共享（NAT/转发）或物理层连接。
- **Wi-Fi 热点（网络共享）**
    - 本质是创建一个软 AP（Access Point），通过宿主机的网络连接（如以太网、4G）共享互联网。
    - 通常依赖 **DHCP + NAT + 网络转发** 的完整套件（例如 `hostapd` + `dnsmasq` + `iptables`）。
    - 除了分配 IP，还会启用 NAT 转换，允许子设备通过宿主机上网。

|**特性**|**dnsmasq (独立 DHCP)**|**Wi-Fi 热点共享**|
|---|---|---|
|**主要功能**|IP 分配 + DNS|IP 分配 + NAT + 互联网共享|
|**网络拓扑**|同局域网无网关|宿主机作为网关和 NAT 设备|
|**配置复杂度**|手动编辑配置文件|依赖自动化工具或多组件协作|
|**典型场景**|本地开发、小型局域网|临时无线网络共享互联网|

根据需求选择：
- 仅需分配 IP → 独立 `dnsmasq`。
    
- 需共享互联网 → Wi-Fi 热点（本质是 `dnsmasq` + NAT 的组合方案）。


# 四、Dnsmasq详解

`dnsmasq` 是一个轻量级、易于配置的，提供 **DNS 缓存**、**DHCP 服务**、**TFTP 服务** 和 **路由器广告（Router Advertisement）** 功能的软件。

## 1.核心功能详解

### （1）DNS 转发与缓存 (DNS Forwarding and Caching)

这是 `dnsmasq` 最常用、最核心的功能。

- **工作原理**：
    1. **监听请求**：`dnsmasq` 在本地设备（如你的电脑、手机）上运行，监听 DNS 查询请求（默认端口 53）。
    2. **查询缓存**：当收到一个 DNS 查询（如 `www.google.com`）时，它首先检查自己的缓存中是否有该域名且记录未过期。如果有，则**立即返回**缓存中的 IP 地址。这是速度最快的方式。
    3. **转发查询**：如果缓存中没有（或记录已过期），`dnsmasq` 会将查询**转发**到配置的上游 DNS 服务器（例如 `8.8.8.8`, `1.1.1.1` 或你的运营商提供的 DNS）。
    4. **缓存并返回**：收到上游 DNS 服务器的响应后，`dnsmasq` 会将结果**缓存**起来（根据记录的 TTL 时间），然后将结果返回给最初发出请求的客户端。
        
- **带来的好处**：
    - **加速网络访问**：对重复的 DNS 查询（比如多人访问同一个网站）响应极快，显著减少 DNS 解析延迟。
    - **减少上游流量**：大量重复查询在本地就被处理，减少了向外网发出的 DNS 请求数量。
    - **离线访问**：即使短暂断网，之前解析过的域名依然可以通过缓存访问（如果 IP 没有变化）。

### （2）DHCP 服务 (Dynamic Host Configuration Protocol)

`dnsmasq` 可以作为一个完整的 DHCP 服务器，为网络中的设备自动分配 IP 地址、网关、DNS 服务器等信息。

- **功能包括**：
    - **IP 地址分配**：从指定的 IP 地址范围内动态分配地址。
    - **静态租约**：根据设备的 MAC 地址为其固定分配同一个 IP 地址，非常有用。
    - **提供网络配置**：告知客户端默认网关、子网掩码、DNS 服务器地址等。
    - **集成优势**：当同时提供 DNS 和 DHCP 服务时，`dnsmasq` 可以自动将 DHCP 分配的主机名（如 `my-laptop`）注册到本地 DNS 中。这样，网络内的其他设备就可以通过 `my-laptop.local` 这样的主机名直接访问该设备，而无需记住其 IP 地址。

### （3）TFTP 服务 (Trivial File Transfer Protocol)

主要用于网络启动（PXE boot）。`dnsmasq` 内置的 TFTP 服务器可以配合 DHCP 服务，帮助无盘工作站或需要网络安装的计算机从服务器上获取启动所需的文件（如 bootloader、内核镜像等）。

### （4）路由器广告 (Router Advertisement)

支持 IPv6 的路由器广告功能，允许 `dnsmasq` 在 IPv6 网络中宣告自己的存在和网络前缀信息。


## 2.配置文件详解

`dnsmasq` 的主要配置文件通常是 `/etc/dnsmasq.conf`。这个文件包含了大量的注释，清晰地解释了每个选项的用途。以下是一些最关键的配置项：
### DNS 相关配置

```
# 指定上游DNS服务器，可以指定多个
server=8.8.8.8
server=1.1.1.1
# 也可以按域名指定特定的DNS服务器，例如所有.google.com的查询都发给8.8.8.8
server=/google.com/8.8.8.8

# 监听地址，指定dnsmasq在哪个网络接口上提供DNS服务
# 127.0.0.1 表示只对本机有效
# 0.0.0.0 表示监听所有接口
listen-address=127.0.0.1
# 如果想让局域网其他设备使用，需要加上本机在内网的IP，或者0.0.0.0
listen-address=192.168.1.1, 127.0.0.1

# 本地域名解析（最重要的功能之一）
# 将主机名映射到IP，格式：<IP地址> <主机名>
address=/example.com/192.168.1.100
# 也可以使用 /etc/hosts 文件
addn-hosts=/etc/hosts
# 是否读取 /etc/hosts 文件
read-ethers

# 指定本地域名，这样在浏览器输入"server"就会自动解析为"server.lan"
domain=lan
# 扩展主机名，输入主机名自动补全域名
expand-hosts

# DNS缓存大小（默认150条记录），增加可以提升大网络的缓存性能
cache-size=1000

# 不加载上游DNS提供的/etc/hosts文件（通常指大量广告域名黑名单）
no-hosts
```
### DHCP 相关配置
```
# 启用DHCP功能
dhcp-range=192.168.1.50,192.168.1.150,255.255.255.0,12h
# 格式：<起始IP>,<结束IP>,<子网掩码>,<租约时间>

# 指定为客户机分配的网关
dhcp-option=3,192.168.1.1
# 指定为客户机分配的DNS服务器（这里就是dnsmasq自己）
dhcp-option=6,192.168.1.1

# 设置静态IP分配（静态租约）
dhcp-host=AA:BB:CC:DD:EE:FF,192.168.1.201,my-pc
# 格式：<MAC地址>,<指定的IP>,<主机名>
```

## 3.DHCP 分配 IP 的基本流程

DHCP 采用 **DORA（Discover-Offer-Request-Acknowledge）** 四个步骤完成 IP 分配：

1. **DHCP Discover**（客户端广播发现 DHCP 服务器）
    - 设备（客户端）接入网络后，发送 `DHCPDISCOVER` 广播包（目标 IP `255.255.255.255`），寻找可用的 DHCP 服务器。
2. **DHCP Offer**（服务器响应可用的 IP 地址）
    - DHCP 服务器（如 `dnsmasq`）收到请求后，从 IP 地址池中选择一个未分配的 IP，并通过 `DHCPOFFER` 包发送给客户端（包含 IP、子网掩码、租期等信息）。
3. **DHCP Request**（客户端确认请求该 IP）
    - 客户端收到 `DHCPOFFER` 后，发送 `DHCPREQUEST` 广播包，确认接受该 IP 地址（可能同时通知其他 DHCP 服务器不再使用它们的 Offer）。
4. **DHCP Acknowledge**（服务器最终确认分配）
    - 服务器收到 `DHCPREQUEST` 后，发送 `DHCPACK` 确认分配，并记录该 IP 已被占用。客户端正式使用该 IP。
        

`external/dnsmasq/`  
这是 `dnsmasq` 的源代码目录，包含完整的 DHCP 和 DNS 服务器实现。





**Settings中有线网开关UI提交**
```
http://git.fjdynamics.com/fjst_linux/amlogic/android-p-20211030/commit/ab1b8c23e8dc8270c7acbae600c50f5fbc26e740
```




