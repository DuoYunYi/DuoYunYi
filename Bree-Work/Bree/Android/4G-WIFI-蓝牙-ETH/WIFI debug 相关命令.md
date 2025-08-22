# wpa_supplicant

wpa_supplicant是一个连接、配置WIFI的工具，它主要包含wpa_supplicant与wpa_cli两个程序。wpa_supplicant是服务端，wap_cli是客户端，一般情况下使用wpa_cli就可以操作WiFi。

### wpa所在路径

```
/etc/wifi/wpa_supplicant.conf

./data/vendor/wifi/wpa/wpa_supplicant.conf
./vendor/etc/wifi/wpa_supplicant.conf
```
![[Pasted image 20250721143852.png]]

### 开启关闭wpa

```
start wpa_supplicant

stop wpa_supplicant
```

### 获取wpa状态

```
adb shell getprop init.svc.wpa_supplicant
```

### 查看wpa相关进程

```
ps -ef | grep wpa_s
```


# wpa_cli

**`wpa_cli`** 是 Linux/Android 系统中用于与 `wpa_supplicant`（Wi-Fi 连接管理服务）交互的命令行工具

### 进入交互模式

adb shell wpa_cli -i wlan0  # 指定网卡（如 wlan0）

| 选项                                           | 说明                                                     |
| -------------------------------------------- | ------------------------------------------------------ |
| scan                                         | 打开后扫描AP                                                |
| scan_results或scan_r                          | 显示扫描结果                                                 |
| status                                       | 列出目前的联网状态                                              |
| list_networks                                | 列出所有备选网络。目前正连接到的网络会标[CURRENT]，禁用的网络会标[DISABLE]。        |
| add_network                                  | 增加一个备选网络，输出新网络的号码（这个号码替代下文的[network_id]）。注意新网络此时是禁用状态。 |
| set_network [network_id] ssid “Your SSID”    | 设置无线网的名称（SSID）                                         |
| set_network [network_id] key_mgmt WPA-PSK    | 设置无线网的加密方式为WPA-PSK/WPA2-PSK                            |
| set_network [network_id] psk “Your Password” | 设置无线网的PSK密码                                            |
| enable_network [network_id]                  | 启用网络。启用后如果系统搜索到了这个网络，就会尝试连接。                           |
| disable_network [network_id]                 | 禁用网络。                                                  |
| save_config                                  | 保存配置。                                                  |

### 启动wpa_cli

```
wpa_cli -i网口 -p socket所在路径
```

例如像我刚才那么调用的话，则用下面命令启动：

```
wpa_cli -iwlan0 -p /data/misc/wifi/sockets/
```

### wpa_cli查看状态

```
wpa_cli status

wpa_cli -i wlan0 status
```

![[Pasted image 20250721194831.png]]

### 查看是否能够正常扫描

```
wpa_cli -i  wlan0 -s /data/vendor/wifi/wpa/sockets scan

wpa_cli -i wlan0 -s /data/vendor/wifi/wpa/sockets scan_result
```


# wlan0

一般来说，android系统里面，打开wifi的时候会加载驱动，关闭则会卸载wifi驱动

### ifconfig  查看wlan0

```
ifconfig wlan0 up
ifconfig wlan0 down
```

```
svc wifi disable
svc wifi enable
```

### 查看socket是否有wlan0

```
ll /data/vendor/wifi/wpa/sockets/
```
![[Pasted image 20250721135632.png]]










