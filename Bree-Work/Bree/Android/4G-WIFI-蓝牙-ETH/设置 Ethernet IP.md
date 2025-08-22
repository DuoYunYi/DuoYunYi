# 设置Framework











**完全可以，而且这是更优雅和系统级的解决方案！** 在 `init.rc` 中配置 Dnsmasq 服务比在应用层启动更好，因为它：

1. **生命周期管理**：由 init 系统管理进程生命周期
    
2. **自动重启**：进程崩溃时自动重启
    
3. **资源控制**：可以配置 cgroups、OOM 调整等
    
4. **权限管理**：通过 seclabel 正确设置 SELinux 上下文
    

---


### 优化后的 init.rc 配置

bash

# 在 init.rc 或 init.{hardware}.rc 中添加

service dnsmasq-eth0 /system/bin/dnsmasq \
    --keep-in-foreground \
    --no-resolv \
    --no-poll \
    --no-hosts \
    --bind-interfaces \
    --except-interface=lo \
    --interface=eth0 \
    --dhcp-range=${persist.dhcp.range.eth0} \
    --dhcp-option=option:router,${persist.dhcp.gateway.eth0} \
    --dhcp-option=option:dns-server,${persist.dhcp.dns.eth0} \
    --dhcp-leasefile=/data/misc/dnsmasq/dnsmasq.leases \
    --pid-file=/data/misc/dnsmasq/dnsmasq.pid \
    --log-dhcp \
    --log-facility=/data/misc/dnsmasq/dnsmasq.log
    class main
    user root
    group root net_admin
    disabled
    oneshot  # 使用 oneshot 因为 dnsmasq 是守护进程
    seclabel u:r:dnsmasq:s0

# 配置重载服务（用于动态更新配置）
service dnsmasq-reload /system/bin/kill -HUP $(cat /data/misc/dnsmasq/dnsmasq.pid)
    class main
    user root
    disabled
    oneshot
    seclabel u:r:dnsmasq:s0

# 触发器配置
on property:persist.dhcp.service.eth0=1
    # 确保目录存在
    mkdir /data/misc/dnsmasq 0770 root root
    start dnsmasq-eth0

on property:persist.dhcp.service.eth0=0
    stop dnsmasq-eth0

on property:persist.dhcp.range.eth0=*
    # 发送 SIGHUP 给 dnsmasq 重新加载配置
    start dnsmasq-reload

on property:persist.dhcp.gateway.eth0=*
    start dnsmasq-reload

on property:persist.dhcp.dns.eth0=*
    start dnsmasq-reload

# 在接口就绪时自动启动
on property:net.eth0.interface=up && property:persist.dhcp.autostart=1
    setprop persist.dhcp.service.eth0 1

---

### 对应的 Java 代码实现

现在你的 Java 代码只需要设置系统属性，init 系统会自动处理服务的启动和配置：

java

public class DnsmasqManager {
    
    private static final String TAG = "DnsmasqManager";
    
    /**
     * 配置并启动 Dnsmasq 服务
     */
    public boolean configureAndStartDnsmasq(String gatewayIp, String dhcpStart, 
                                          String dhcpEnd, String dnsServers) {
        try {
            // 1. 首先设置静态IP（你的现有代码）
            if (!configureStaticIp(gatewayIp)) {
                return false;
            }
            
            // 2. 设置Dnsmasq配置参数
            String dhcpRange = dhcpStart + "," + dhcpEnd + ",255.255.255.0,12h";
            SystemProperties.set("persist.dhcp.range.eth0", dhcpRange);
            SystemProperties.set("persist.dhcp.gateway.eth0", gatewayIp);
            SystemProperties.set("persist.dhcp.dns.eth0", dnsServers);
            
            // 3. 等待属性设置完成
            Thread.sleep(100);
            
            // 4. 触发服务启动
            SystemProperties.set("persist.dhcp.service.eth0", "1");
            
            Log.d(TAG, "Dnsmasq configuration sent to init system");
            return true;
            
        } catch (Exception e) {
            Log.e(TAG, "Error configuring Dnsmasq: " + e.getMessage(), e);
            return false;
        }
    }
    
    /**
     * 停止 Dnsmasq 服务
     */
    public boolean stopDnsmasq() {
        try {
            SystemProperties.set("persist.dhcp.service.eth0", "0");
            return true;
        } catch (Exception e) {
            Log.e(TAG, "Error stopping Dnsmasq: " + e.getMessage(), e);
            return false;
        }
    }
    
    /**
     * 更新 DHCP 范围
     */
    public boolean updateDhcpRange(String dhcpStart, String dhcpEnd) {
        try {
            String dhcpRange = dhcpStart + "," + dhcpEnd + ",255.255.255.0,12h";
            SystemProperties.set("persist.dhcp.range.eth0", dhcpRange);
            return true;
        } catch (Exception e) {
            Log.e(TAG, "Error updating DHCP range: " + e.getMessage(), e);
            return false;
        }
    }
    
    /**
     * 检查 Dnsmasq 服务状态
     */
    public boolean isDnsmasqRunning() {
        try {
            // 检查pid文件是否存在且进程在运行
            File pidFile = new File("/data/misc/dnsmasq/dnsmasq.pid");
            if (!pidFile.exists()) {
                return false;
            }
            
            // 读取pid并检查进程
            String pid = readFileToString(pidFile.getAbsolutePath()).trim();
            Process process = Runtime.getRuntime().exec("ps " + pid);
            BufferedReader reader = new BufferedReader(
                new InputStreamReader(process.getInputStream()));
            
            String line;
            while ((line = reader.readLine()) != null) {
                if (line.contains("dnsmasq")) {
                    return true;
                }
            }
            
            return false;
            
        } catch (Exception e) {
            Log.e(TAG, "Error checking Dnsmasq status: " + e.getMessage());
            return false;
        }
    }
    
    /**
     * 获取当前租约信息
     */
    public String getDhcpLeases() {
        try {
            File leasesFile = new File("/data/misc/dnsmasq/dnsmasq.leases");
            if (leasesFile.exists()) {
                return readFileToString(leasesFile.getAbsolutePath());
            }
            return "No leases found";
        } catch (Exception e) {
            return "Error reading leases: " + e.getMessage();
        }
    }
    
    private String readFileToString(String path) throws IOException {
        BufferedReader reader = new BufferedReader(new FileReader(path));
        StringBuilder content = new StringBuilder();
        String line;
        while ((line = reader.readLine()) != null) {
            content.append(line).append("\n");
        }
        reader.close();
        return content.toString();
    }
    
    private boolean configureStaticIp(String gatewayIp) {
        // 你的现有静态IP配置代码
        // ...
        return true;
    }
}

---

### 需要的 SELinux 策略

在 `dnsmasq.te` 文件中添加：

selinux

# Dnsmasq SELinux policy
type dnsmasq, domain;
type dnsmasq_exec, exec_type, file_type;

init_daemon_domain(dnsmasq)

# 允许读写租约文件和pid文件
allow dnsmasq system_data_file:dir { search create };
allow dnsmasq system_data_file:file { create read write open unlink };

# 允许网络操作
allow dnsmasq self:udp_socket create_socket_perms;
allow dnsmasq self:rawip_socket create_socket_perms;
allow dnsmasq port:dns_port name_bind;
allow dnsmasq port:dhcp_server_port name_bind;

# 允许访问系统属性
allow dnsmasq system_prop:property_service set;

---

### 测试和验证

java

// 测试代码
DnsmasqManager manager = new DnsmasqManager();

// 启动服务
boolean success = manager.configureAndStartDnsmasq(
    "192.168.1.1",         // gateway
    "192.168.1.100",       // dhcp start
    "192.168.1.200",       // dhcp end
    "8.8.8.8,8.8.4.4"      // dns servers
);

if (success) {
    // 检查状态
    if (manager.isDnsmasqRunning()) {
        Log.d(TAG, "Dnsmasq is running");
        
        // 等待一段时间后查看租约
        new Handler().postDelayed(() -> {
            String leases = manager.getDhcpLeases();
            Log.d(TAG, "DHCP Leases:\n" + leases);
        }, 5000);
    }
}