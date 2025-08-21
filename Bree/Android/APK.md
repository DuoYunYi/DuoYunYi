# Android RecyclerView表格布局



# udp




# http

HTTP协议是Hyper Text Transfer Protocol（超文本传输协议）的缩写,是用于从万维网（WWW:World Wide Web ）服务器传输超文本到本地浏览器的传送协议。

HTTP是一个基于TCP/IP通信协议来传递数据（HTML 文件, 图片文件, 查询结果等）。

HTTP是一个属于应用层的面向对象的协议，由于其简捷、快速的方式，适用于分布式超媒体信息系统。

HTTP协议工作于客户端-服务端架构为上。浏览器作为HTTP客户端通过URL向HTTP服务端即WEB服务器发送所有请求。Web服务器根据接收到的请求后，向客户端发送响应信息。

主要特点：简单快速、灵活、无连接、无状态、支持B/S及C/S模式


URI，是uniform resource identifier，统一资源标识符，用来唯一的标识一个资源。

URL是uniform resource locator，统一资源定位器，它是一种具体的URI，即URL可以用来标识一个资源，而且还指明了如何locate这个资源。


**HTTP之请求消息Request**

请求消息包括以下格式：

**请求行（request line）、请求头部（header）、空行和请求数据**四个部分组成。

**HTTP之响应消息Response**

HTTP响应也由四个部分组成，分别是：**状态行、消息报头、空行和响应正文**。


常见状态码：
```
200 OK                        //客户端请求成功
400 Bad Request               //客户端请求有语法错误，不能被服务器所理解
401 Unauthorized              //请求未经授权，这个状态代码必须和WWW-Authenticate报头域一起使用 
403 Forbidden                 //服务器收到请求，但是拒绝提供服务
404 Not Found                 //请求资源不存在，eg：输入了错误的URL
500 Internal Server Error     //服务器发生不可预期的错误
503 Server Unavailable        //服务器当前不能处理客户端的请求，一段时间后可能恢复正常
```

工作原理：
HTTP协议采用了请求/响应模型。客户端向服务器发送一个请求报文，请求报文包含请求的方法、URL、协议版本、请求头部和请求数据。服务器以一个状态行作为响应，响应的内容包括协议的版本、成功或者错误代码、服务器信息、响应头部和响应数据。



socket.setReceiveBufferSize(1024 * 1024); // 1MB Log.d("UDP", "缓冲区大小: " + socket.getReceiveBufferSize()); 作用

`socket.setReceiveBufferSize(1024 * 1024)` 和 `socket.getReceiveBufferSize()` 是 Java 中用于控制 UDP 接收缓冲区大小的关键方法，它们的作用和实际效果如下：

---

### **1. `setReceiveBufferSize(1024 * 1024)` 的作用**

- **功能**：设置 UDP Socket 的接收缓冲区大小为 **1MB**（1024 × 1024 字节）。
    
- **为什么需要调整**：
    
    - 默认缓冲区大小（通常约 64KB~256KB）可能无法应对高流量或突发数据。
        
    - 如果数据包到达速度过快，而缓冲区已满，新数据包会被**静默丢弃**（UDP 的不可靠特性）。
        
    - 增大缓冲区可以减少因系统繁忙导致的丢包概率。
        
- **注意事项**：
    
    - 实际生效值可能受操作系统限制（最终大小可能小于设置值）。
        
    - 设置后需通过 `getReceiveBufferSize()` 验证实际值。
        

---

### **2. `getReceiveBufferSize()` 的作用**

- **功能**：获取当前 Socket 的**实际接收缓冲区大小**（单位：字节）。
    
- **为什么需要调用**：
    
    - 操作系统可能会根据内核参数自动调整用户设置的值（例如限制最大值）。
        
    - 通过日志输出实际值，可以确认设置是否生效。
        

---

### **3. 实际应用场景**

#### **案例：高速 UDP 数据传输**

java

DatagramSocket socket = new DatagramSocket(PORT);
socket.setReceiveBufferSize(1024 * 1024); // 尝试设置为 1MB
Log.d("UDP", "实际缓冲区大小: " + socket.getReceiveBufferSize());

byte[] buffer = new byte[8192];
DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
socket.receive(packet); // 从缓冲区读取数据

- **日志输出**：
    
    text
    

- 实际缓冲区大小: 1048576  // 成功设置为 1MB
    或
    实际缓冲区大小: 65536   // 操作系统未接受设置，保留默认值
    

---

### **4. 系统级限制与调优**

- **Linux/Android 系统**：
    
    - 通过 `sysctl` 查看内核 UDP 缓冲区默认最大值：
        
        bash
        

- sysctl net.core.rmem_max
    
- 如果需要更大的缓冲区，需修改系统参数（需 root 权限）：
    
    bash
    

- - sysctl -w net.core.rmem_max=1048576
        
- **Windows 系统**：
    
    - 通过注册表调整 `HKLM\SYSTEM\CurrentControlSet\Services\AFD\Parameters\DefaultReceiveWindow`。
        

---

### **5. 常见问题**

- **设置不生效**：
    
    - 如果 `getReceiveBufferSize()` 返回值与设置值不符，说明被操作系统限制。
        
    - 需按上述方法调整系统级参数。
        
- **缓冲区过大**：
    
    - 过大的缓冲区会增加内存占用，可能导致延迟（数据在缓冲区中排队）。
        
    - 需权衡吞吐量和实时性。
        

---

### **总结**

|方法|作用|典型值|注意事项|
|---|---|---|---|
|`setReceiveBufferSize()`|设置期望的缓冲区大小|1MB~8MB|受系统限制|
|`getReceiveBufferSize()`|获取实际缓冲区大小|依赖系统|用于调试验证|

如果仍有 UDP 丢包问题，需结合：

1. 检查系统内核参数。
    
2. 优化接收代码逻辑（如缩短处理时间）。
    
3. 网络抓包（Wireshark）确认数据是否到达网卡。





| 特性       | 应用程序层 `buf`                    | 操作系统 Socket 接收缓冲区                    |
| -------- | ------------------------------ | ------------------------------------ |
| **作用**   | 存储单次 `receive()` 调用的数据包        | 缓存尚未被应用层读取的**所有待处理数据包**              |
| **大小控制** | 由 `byte[] buf` 长度决定（如 65507）   | 由 `socket.setReceiveBufferSize()` 设置 |
| **溢出后果** | 数据截断（需检查 `packet.getLength()`） | 数据包丢失（操作系统直接丢弃）                      |
| **典型场景** | 处理单次数据包内容                      | 应对高流量或突发数据                           |