

> 平时常用的springcloud等框架之间的调用用的是http协议，该协议通信是基于tcp/ip之上的应用层协议。其他的应用层协议还有：DNS、Telnet等

## 网络模型

<img src="TCP:IP%E5%8D%8F%E8%AE%AE.assets/image-20210309101540660.png" alt="image-20210309101540660" style="zoom:50%;" />

### tcp/ip四层概念模型

使用tcp进行传输，逐个通过每一层（比特流传输），每一层增加一些信息<img src="TCP:IP%E5%8D%8F%E8%AE%AE.assets/image-20210309104140326.png" alt="image-20210309104140326" style="zoom:50%;" />



**数据链路层实现了网卡驱动程序，让数据在（以太网等）上传输**

数据链路层常用的协议是ARP协议：通过已知的目标ip的地址转换为物理地址（mac地址）

**原理：**通过广播。对应的机器会认领该ip，获取该机器的mac地址

ARP协议：存在`缓存`，ip可能变化，一段时间后缓存过期



### 负载均衡

#### 二层负载（数据链路层）

基于虚拟的mac地址，转发的真实的mac地址

#### 三层负载（网络层）

基于虚拟ip地址接收请求，转发到真实的ip地址

#### 四层负载（传输层）

基于虚拟ip+port地址接收请求，转发到真实的ip+port地址

#### 七层负载（应用层）

基于虚拟url地址接收请求，转发到真实的url地址



### tcp握手协议

https://yuanrengu.blog.csdn.net/article/details/102366854



### io多路复用

https://zhuanlan.zhihu.com/p/126278747