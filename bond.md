# bonding技术

bonding(绑定)是一种linux系统下的网卡绑定技术，可以把服务器上n个物理网卡在系统内部抽象(绑定)成一个逻辑上的网卡，能够提升网络吞吐量、实现网络冗余、负载等功能，有很多优势。
bonding技术是linux系统内核层面实现的，它是一个内核模块(驱动)。使用它需要系统有这个模块, 我们可以modinfo命令查看下这个模块的信息, 一般来说都支持.

---

## bonding的七种工作模式:

bonding技术提供了七种工作模式，在使用的时候需要指定一种，每种有各自的优缺点.

balance-rr (mode=0)       默认, 有高可用 (容错) 和负载均衡的功能,  需要交换机的配置，每块网卡轮询发包 (流量分发比较均衡).
active-backup (mode=1)  只有高可用 (容错) 功能, 不需要交换机配置, 这种模式只有一块网卡工作, 对外只有一个mac地址。缺点是端口利用率比较低
balance-xor (mode=2)     不常用
broadcast (mode=3)        不常用
802.3ad (mode=4)          IEEE 802.3ad 动态链路聚合，需要交换机配置，没用过
balance-tlb (mode=5)      不常用
balance-alb (mode=6)     有高可用 ( 容错 )和负载均衡的功能，不需要交换机配置  (流量分发到每个接口不是特别均衡)
具体的网上有很多资料，了解每种模式的特点根据自己的选择就行, 一般会用到0、1、4、6这几种模式。

### 更为详细的说明

1、mode=0(balance-rr)(平衡抡循环策略)
链路负载均衡，增加带宽，支持容错，一条链路故障会自动切换正常链路。交换机需要配置聚合口，思科叫port channel。
特点：传输数据包顺序是依次传输（即：第1个包走eth0，下一个包就走eth1….一直循环下去，直到最后一个传输完毕），此模式提供负载平衡和容错能力；但是我们知道如果一个连接
或者会话的数据包从不同的接口发出的话，中途再经过不同的链路，在客户端很有可能会出现数据包无序到达的问题，而无序到达的数据包需要重新要求被发送，这样网络的吞吐量就会下降
2、mode=1(active-backup)(主-备份策略)
这个是主备模式，只有一块网卡是active，另一块是备用的standby，所有流量都在active链路上处理，交换机配置的是捆绑的话将不能工作，因为交换机往两块网卡发包，有一半包是丢弃的。
特点：只有一个设备处于活动状态，当一个宕掉另一个马上由备份转换为主设备。mac地址是外部可见得，从外面看来，bond的MAC地址是唯一的，以避免switch(交换机)发生混乱。
此模式只提供了容错能力；由此可见此算法的优点是可以提供高网络连接的可用性，但是它的资源利用率较低，只有一个接口处于工作状态，在有 N 个网络接口的情况下，资源利用率为1/N
3、mode=2(balance-xor)(平衡策略)
表示XOR Hash负载分担，和交换机的聚合强制不协商方式配合。（需要xmit_hash_policy，需要交换机配置port channel）
特点：基于指定的传输HASH策略传输数据包。缺省的策略是：(源MAC地址 XOR 目标MAC地址) % slave数量。其他的传输策略可以通过xmit_hash_policy选项指定，此模式提供负载平衡和容错能力
4、mode=3(broadcast)(广播策略)
表示所有包从所有网络接口发出，这个不均衡，只有冗余机制，但过于浪费资源。此模式适用于金融行业，因为他们需要高可靠性的网络，不允许出现任何问题。需要和交换机的聚合强制不协商方式配合。
特点：在每个slave接口上传输每个数据包，此模式提供了容错能力
5、mode=4(802.3ad)(IEEE 802.3ad 动态链接聚合)
表示支持802.3ad协议，和交换机的聚合LACP方式配合（需要xmit_hash_policy）.标准要求所有设备在聚合操作时，要在同样的速率和双工模式，而且，和除了balance-rr模式外的其它bonding负载均衡模式一样，任何连接都不能使用多于一个接口的带宽。
特点：创建一个聚合组，它们共享同样的速率和双工设定。根据802.3ad规范将多个slave工作在同一个激活的聚合体下。
外出流量的slave选举是基于传输hash策略，该策略可以通过xmit_hash_policy选项从缺省的XOR策略改变到其他策略。需要注意的 是，并不是所有的传输策略都是802.3ad适应的，
尤其考虑到在802.3ad标准43.2.4章节提及的包乱序问题。不同的实现可能会有不同的适应 性。
必要条件：
条件1：ethtool支持获取每个slave的速率和双工设定
条件2：switch(交换机)支持IEEE 802.3ad Dynamic link aggregation
条件3：大多数switch(交换机)需要经过特定配置才能支持802.3ad模式
6、mode=5(balance-tlb)(适配器传输负载均衡)
是根据每个slave的负载情况选择slave进行发送，接收时使用当前轮到的slave。该模式要求slave接口的网络设备驱动有某种ethtool支持；而且ARP监控不可用。
特点：不需要任何特别的switch(交换机)支持的通道bonding。在每个slave上根据当前的负载（根据速度计算）分配外出流量。如果正在接受数据的slave出故障了，另一个slave接管失败的slave的MAC地址。
必要条件：
ethtool支持获取每个slave的速率
7、mode=6(balance-alb)(适配器适应性负载均衡)
在5的tlb基础上增加了rlb(接收负载均衡receive load balance).不需要任何switch(交换机)的支持。接收负载均衡是通过ARP协商实现的.
特点：该模式包含了balance-tlb模式，同时加上针对IPV4流量的接收负载均衡(receive load balance, rlb)，而且不需要任何switch(交换机)的支持。接收负载均衡是通过ARP协商实现的。bonding驱动截获本机发送的ARP应答，并把源硬件地址改写为bond中某个slave的唯一硬件地址，从而使得不同的对端使用不同的硬件地址进行通信。
来自服务器端的接收流量也会被均衡。当本机发送ARP请求时，bonding驱动把对端的IP信息从ARP包中复制并保存下来。当ARP应答从对端到达 时，bonding驱动把它的硬件地址提取出来，并发起一个ARP应答给bond中的某个slave。
使用ARP协商进行负载均衡的一个问题是：每次广播 ARP请求时都会使用bond的硬件地址，因此对端学习到这个硬件地址后，接收流量将会全部流向当前的slave。这个问题可以通过给所有的对端发送更新 （ARP应答）来解决，应答中包含他们独一无二的硬件地址，从而导致流量重新分布。
当新的slave加入到bond中时，或者某个未激活的slave重新 激活时，接收流量也要重新分布。接收的负载被顺序地分布（round robin）在bond中最高速的slave上
当某个链路被重新接上，或者一个新的slave加入到bond中，接收流量在所有当前激活的slave中全部重新分配，通过使用指定的MAC地址给每个 client发起ARP应答。下面介绍的updelay参数必须被设置为某个大于等于switch(交换机)转发延时的值，从而保证发往对端的ARP应答 不会被switch(交换机)阻截。
必要条件：
条件1：ethtool支持获取每个slave的速率；
条件2：底层驱动支持设置某个设备的硬件地址，从而使得总是有个slave(curr_active_slave)使用bond的硬件地址，同时保证每个bond 中的slave都有一个唯一的硬件地址。如果curr_active_slave出故障，它的硬件地址将会被新选出来的 curr_active_slave接管
其实mod=6与mod=0的区别：mod=6，先把eth0流量占满，再占eth1，….ethX；而mod=0的话，会发现2个口的流量都很稳定，基本一样的带宽。而mod=6，会发现第一个口流量很高，第2个口只占了小部分流量。
 
mode5和mode6不需要交换机端的设置，网卡能自动聚合。mode4需要支持802.3ad。mode0，mode2和mode3理论上需要静态聚合方式。
但实测中mode0可以通过mac地址欺骗的方式在交换机不设置的情况下不太均衡地进行接收。


---

# Centos7配置bonding

服务器上两张物理网卡em1和em2, 通过绑定成一个逻辑网卡bond0，bonding模式选择mode6

注: ip地址配置在bond0上, 物理网卡不需要配置ip地址.

1、关闭和停止NetworkManager服务

```
systemctl stop NetworkManager.service     # 停止NetworkManager服务
systemctl disable NetworkManager.service  # 禁止开机启动NetworkManager服务

```

2、加载bonding模块

```
modprobe --first-time bonding

```

没有提示说明加载成功, 如果出现modprobe: ERROR: could not insert 'bonding': Module already in kernel说明你已经加载了这个模块, 就不用管了
你也可以使用lsmod | grep bonding查看模块是否被加载

3、创建基于bond0接口的配置文件

```
vim /etc/sysconfig/network-scripts/ifcfg-bond0

```

修改成如下，根据你的情况:

```
DEVICE=bond0
TYPE=Bond
IPADDR=172.16.0.183
NETMASK=255.255.255.0
GATEWAY=172.16.0.1
DNS1=114.114.114.114
USERCTL=no
BOOTPROTO=none
ONBOOT=yes
BONDING_MASTER=yes
BONDING_OPTS="mode=6 miimon=100"
```

上面的BONDING_OPTS="mode=6 miimon=100" 表示这里配置的工作模式是mode6(adaptive load balancing), miimon表示监视网络链接的频度 (毫秒), 我们设置的是100毫秒, 根据你的需求也可以指定mode成其它的负载模式。

4、修改em1接口的配置文件

```
vim /etc/sysconfig/network-scripts/ifcfg-em1
```

修改成如下:

```
DEVICE=em1
USERCTL=no
ONBOOT=yes
MASTER=bond0                  # 需要和上面的ifcfg-bond0配置文件中的DEVICE的值对应
SLAVE=yes
BOOTPROTO=none
```

5、修改em2接口的配置文件

```
vim /etc/sysconfig/network-scripts/ifcfg-em2
```

修改成如下:

```
DEVICE=em2
USERCTL=no
ONBOOT=yes
MASTER=bond0                  # 需要和上面的ifcfg-bond0配置文件中的DEVICE的值对应
SLAVE=yes
BOOTPROTO=none
```

6、测试
重启网络服务
```
systemctl restart network
```

查看bond0的接口状态信息  ( 如果报错说明没做成功，很有可能是bond0接口没起来)

```
# cat /proc/net/bonding/bond0

Bonding Mode: adaptive load balancing   // 绑定模式: 当前是ald模式(mode 6), 也就是高可用和负载均衡模式
Primary Slave: None
Currently Active Slave: em1
MII Status: up                           // 接口状态: up(MII是Media Independent Interface简称, 接口的意思)
MII Polling Interval (ms): 100           // 接口轮询的时间隔(这里是100ms)
Up Delay (ms): 0
Down Delay (ms): 0

Slave Interface: em1                     // 备接口: em0
MII Status: up                           // 接口状态: up(MII是Media Independent Interface简称, 接口的意思)
Speed: 1000 Mbps                         // 端口的速率是1000 Mpbs
Duplex: full                             // 全双工
Link Failure Count: 0                    // 链接失败次数: 0 
Permanent HW addr: 84:2b:2b:6a:76:d4      // 永久的MAC地址
Slave queue ID: 0

Slave Interface: em1                     // 备接口: em1
MII Status: up                           // 接口状态: up(MII是Media Independent Interface简称, 接口的意思)
Speed: 1000 Mbps
Duplex: full                             // 全双工
Link Failure Count: 0                    // 链接失败次数: 0
Permanent HW addr: 84:2b:2b:6a:76:d5     // 永久的MAC地址
Slave queue ID: 0
```
通过ifconfig命令查看下网络的接口信息

```
# ifconfig

bond0: flags=5187<UP,BROADCAST,RUNNING,MASTER,MULTICAST>  mtu 1500
        inet 172.16.0.183  netmask 255.255.255.0  broadcast 172.16.0.255
        inet6 fe80::862b:2bff:fe6a:76d4  prefixlen 64  scopeid 0x20<link>
        ether 84:2b:2b:6a:76:d4  txqueuelen 0  (Ethernet)
        RX packets 11183  bytes 1050708 (1.0 MiB)
        RX errors 0  dropped 5152  overruns 0  frame 0
        TX packets 5329  bytes 452979 (442.3 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

em1: flags=6211<UP,BROADCAST,RUNNING,SLAVE,MULTICAST>  mtu 1500
        ether 84:2b:2b:6a:76:d4  txqueuelen 1000  (Ethernet)
        RX packets 3505  bytes 335210 (327.3 KiB)
        RX errors 0  dropped 1  overruns 0  frame 0
        TX packets 2852  bytes 259910 (253.8 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

em2: flags=6211<UP,BROADCAST,RUNNING,SLAVE,MULTICAST>  mtu 1500
        ether 84:2b:2b:6a:76:d5  txqueuelen 1000  (Ethernet)
        RX packets 5356  bytes 495583 (483.9 KiB)
        RX errors 0  dropped 4390  overruns 0  frame 0
        TX packets 1546  bytes 110385 (107.7 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 0  (Local Loopback)
        RX packets 17  bytes 2196 (2.1 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 17  bytes 2196 (2.1 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

测试网络高可用, 我们拔掉其中一根网线进行测试, 结论是：

在本次mode=6模式下丢包1个, 恢复网络时( 网络插回去 ) 丢包在5-6个左右，说明高可用功能正常但恢复的时候丢包会比较多
测试mode=1模式下丢包1个，恢复网络时( 网线插回去 ) 基本上没有丢包，说明高可用功能和恢复的时候都正常
mode6这种负载模式除了故障恢复的时候有丢包之外其它都挺好的，如果能够忽略这点的话可以这种模式；而mode1故障的切换和恢复都很快，基本没丢包和延时。但端口利用率比较低，因为这种主备的模式只有一张网卡在工作.
