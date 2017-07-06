# Linux的线程理解

## Linux线程的定义
线程（thread）是在共享内存空间中并发的多道执行路径，它们共享一个进程的资源，如文件描述和信号处理。
在两个普通进程（非线程）间进行切换时，内核准备从一个进程的上下文切换到另一个进程的上下文要花费很大的开销。
这里上下文切换的主要任务是保存老进程CPU状态并加载新进程的保存状态，用新进程的内存映像替换进程的内存映像。
线程允许你的进程在几个正在运行的任务之间进行切换，而不必执行前面提到的完整的上下文。另外本文介绍的线程是针对POSIX线程，也就是pthread。
也因为Linux对它的支持最好。相对进程而言，线程是一个更加接近于执行体的概念，它可以与同进程中的其他线程共享数据，但拥有自己的栈空间，拥有独立的执行序列。
在串行程序基础上引入线程和进程是为了提高程序的并发度，从而提高程序运行效率和响应时间。也可以将线程和轻量级进程（LWP）视为等同的，但其实在不同的系统/实现中有不同的解释，LWP更恰当的解释为一个虚拟CPU或内核的线程。
它可以帮助用户态线程实现一些特殊的功能。Pthread是一种标准化模型，它用来把一个程序分成一组能够同时执行的任务。
在检查程序中潜在的并行性时，也就是说在要找出能够同时执行任务时使用Pthread。
上面已经介绍了，Linux进程模型提供了执行多个进程的能力，已经可以进行并行或并发编程，可是线程能够让你对多个任务的控制程度更好、使用资源更少，因为一个单一的资源，如全局变量，可以由多个线程共享。而且，在拥有多个处理器的系统上，多线程应用会比用多个进程实现的应用执行速度更快。

---

## Linux进程和线程的发展

1999年1月发布的Linux 2.2内核中，进程是通过系统调用fork创建的，新的进程是原来进程的子进程。需要说明的是，在2.2.x中，不存在真正意义上的线程（Thread）。
Linux中常用的线程Pthread实际上是通过进程来模拟的。也就是说Linux中的线程也是通过fork创建的，是"轻"进程。
Linux 2.2默认只允许4096个进程/线程同时运行。高端系统同时要服务上千的用户，所以这显然是一个问题，一度是阻碍Linux进入企业级市场的一大因素。
2001年1月发布的Linux 2.4内核消除了这个限制，并且允许在系统运行中动态调整进程数上限。
因此，进程数现在只受制于物理内存的多少。在高端服务器上，即使只安装了512MB内存，现在也能轻而易举地同时支持1万6千个进程。
2003年12月发布的2.6内核，进程调度经过重新编写，去掉了以前版本中效率不高的算法。以前，为了决定下一步要运行哪一个任务，进程调度程序要查看每一个准备好的任务，并且经过计算来决定哪一个任务相对来说更为重要。
进程标识号（PID）的数目也从32000升到10亿。内核内部的大改变之一就是Linux的线程框架被重写，以使NPTL（Native POSIX Thread Library）可以运行于其上。
对于运行负荷繁重的线程应用的Pentium Pro及更先进的处理器而言，这是一个主要的性能提升，也是企业级应用中的很多高端系统一直以来所期待的。
线程框架的改变包含Linux线程空间中的许多新的概念，包括线程组、线程各自的本地存储区、POSIX风格信号，以及其他改变。改进后的多线程和内存管理技术有助于更好地运行大型多媒体应用软件。

---

## 进程与线程的优缺点

线程和进程在使用上各有优缺点：线程执行开销小，但不利于资源的管理和保护；而进程正相反。同时，线程适合于在对称多处理器的计算机上运行，而进程则可以跨机器迁移。另外进程可以拥有资源，线程共享进程拥有的资源。
进程间的切换必须保存在进程控制块PCB（Process Control Block）中。同个进程的多个线程间的切换不用那么麻烦。

---

## Linux守护进程

进程一般分为交互进程、批处理进程和守护进程（daemons）三类。守护进程总是活跃的，一般在后台运行，守护进程一般由系统在开机时通过脚本自动激活启动或由超级管理用户root来启动。例如httpd服务。
由于守护进程是一直运行着的，所以它所处的状态是等待请求处理任务。例如通常大网站的Apache服务器都在运行，等待着用户来访问，也就是等待着任务处理。

### 100个最常见Linux守护进程简介(部分内容较老，可能最新的版本中不存在了)

1．alsasound：Alsa声卡驱动守护程序。Alsa声卡驱动程序本来是为了一种声卡Gravis UltraSound（GUS)而写的，该程序被证明很优秀，于是作者就开始为一般的声卡写驱动程序。Alsa和OSS/Free及OSS/Linux兼容，但是有自己的接口，甚至比OSS优秀。
2．acpid：acpid（Advanced Configuration and Power Interface）是为替代传统的APM电源管理标准而推出的新型电源管理标准。通常笔记本电脑需要启动电源进行管理。
3．atalk：AppleTalk网络守护进程。注意不要在后台运行该程序，该程序的数据结构必须在运行其他进程前先花一定时间初始化。
4．amd：自动安装NFS守护进程。
5．anacron：一个自动化运行任务守护进程。Red Hat Linux随带四个自动化任务的工具：cron、anacron、at和batc。当你的Linux服务器并不是全天运行时，这个anacron就可以帮你执行在"crontab"设定的时间内没有执行的工作。
6．apmd：apmd（Advanced Power Management）是高级电源管理。传统的电源管理标准，对于笔记本电脑比较有用，可以了解系统的电池电量信息。并将相关信息通过syslogd写入日志。也可以用来在电源不足时关机。
7．arptables_jf：为arptables网络的用户控制过滤的守护进程。
8．arpwatch：记录日志并构建一个在LAN接口上看到的以太网地址和IP地址对数据库。
9．atd：at和batch命令守护进程，用户用at命令调度的任务。batch用于在系统负荷比较低时运行批处理任务。
10．autofs：自动安装管理进程automount，与NFS相关，依赖于NIS服务器。
11．bootparamd：引导参数服务器，为LAN上的无盘工作站提供引导所需的相关信息。
12．bluetooch：蓝牙服务器守护进程。
13．crond：cron是UNIX下的一个传统程序，该程序周期性地运行用户调度的任务。比起传统的UNIX版本，Linux版本添加了不少属性，而且更安全，配置更简单。类似于计划任务。
14．chargen：使用tcp协议的chargen server，chargen（Character Generator Protocol）是一种网络服务，主要功能是提供类似于远程打字的功能。
15．chargen-udp：使用UDP协议的chargen server。
16．cpuspeed：监测系统空闲百分比，降低或加快CPU时钟速度和电压，从而在系统空闲时将能源消耗降为最小，而在系统繁忙时最大化加快系统执行速度。
17．dhcpd：动态主机控制协议（Dynamic Host Control Protocol）的服务守护进程。
18．cups：cups(Common UNIX Printing System)是通用UNIX打印守护进程，为Linux提供第三代打印功能。
19．cups－config－daemons：cups打印系统切换守护进程。
20．cups-lpd：cups行打印守护进程。
21．daytime：使用TCP协议的Daytime守护进程，该协议为客户机实现从远程服务器获取日期和时间的功能。预设端口：13。
22．daytime-udp：使用UDP协议的Daytime守护进程。
23．dc_server：使用SSL安全套接字的代理服务器守护进程。
24．dc_client：使用SSL安全套接字的客户端守护进程。
25．diskdump：服务器磁盘备份守护进程。
26．echo：服务器回显客户数据服务守护进程。
27．echo-udp：使用UDP协议的服务器回显客户数据服务守护进程。
28．eklogin：接受rlogin会话鉴证和用kerberos5加密的一种服务的守护进程。
29．gated：网关路由守护进程。它支持各种路由协议，包括RIP版本1和2、DCN HELLO协议、OSPF版本2，以及EGP版本2到4。
30．gpm：gpm（General Purpose Mouse Daemon）守护进程为文本模式下的Linux程序如mc(Midnight Commander)提供了鼠标的支持。它也支持控制台下鼠标的复制、粘贴操作，以及弹出式菜单。
31．gssftp：使用kerberos 5认证的FTP守护进程。
32．httpd：Web服务器Apache守护进程，可用来提供HTML文件及CGI动态内容服务。
33．inetd：因特网操作守护程序。监控网络对各种它管理的服务的需求，并在必要的时候启动相应的服务程序。在Redhat和Mandrake linux中被xinetd代替。Debian，Slackware，SuSE仍然使用。
34．innd：Usenet新闻服务器守护进程。
35．iiim：中文输入法服务器守护进程。
36．iptables：iptables防火墙守护进程。
37．irda：红外端口守护进程。
38．isdn：isdn启动和中止服务守护进程。
39．krb5－telnet：使用kerberos 5认证的Telnet守护进程。
40．klogin：远程登录守护进程。
41．keytable：该进程的功能是转载在/etc/sysconfig/keyboards里定义的键盘映射表，该表可以通过kbdconfig工具进行选择。你应该使该程序处于激活状态。
42．irqbalance：对多个系统处理器环境下的系统中断请求进行负载平衡的守护程序。如果你只安装了一个CPU，就不需要加载这个守护程序。
43．kshell：kshell守护进程。
44．kudzu：硬件自动检测程序，会自动检测硬件是否发生变动，并相应进行硬件的添加、删除工作。当系统启动时，kudzu会对当前的硬件进行检测，并且和存储在/etc/sysconfig/hwconf中的硬件信息进行对照，如果某个硬件从系统中被添加或者删除时，那么kudzu就会察觉到，并且通知用户是否进行相关配置，然后修改etc/sysconfig/hwconf，使硬件资料与系统保持同步。如果/etc/sysconfig/hwconf这个文件不存在，那么kudzu将会从/etc/modprobe.conf，/etc/sysconfig/network-scripts/和etc/X11/XF86Config中探测已经存在的硬件。如果你不打算增加新硬件，那么就可以关闭这个启动服务，以加快系统启动时间。
45．ldap：ldap（Lightweight Directory Access Protocol）目录访问协议服务器守护进程。
46．lm_seroems：检测主板工作情况守护进程。
47．lpd：lpd是老式打印守护程序，负责将lpr等程序提交给打印作业。
48．mdmonitor：RAID相关设备的守护程序。
49．messagebus：D-BUS是一个库，为两个或两个以上的应用程序提供一对一的通信。dbus-daemon-1是一个应用程序，它使用这个库来实现messagebus守护程序。多个应用程序通过连接messagebus守护程序可以实现与其他程序交换信息。
50．microcode_ctl：可编码及发送新的微代码到内核以更新Intel IA32系列处理器守护进程。
51．mysqld：一个快速、高效、可靠的轻型SQL数据库引擎守护进程。
52．named：DNS（BIND）服务器守护进程。
53．netplugd：netplugd（network cable hotplug management daemon）守护程序，用于监控一个或多个网络接口的状态，当某些事件触发时运行一个外部脚本程序。
54．netdump：远程网络备份服务器守护进程。
55．netfs：Network Filesystem Mounter，该进程安装和卸载NFS、SAMBA和NCP网络文件系统。
56．nfs：网络文件系统守护进程。
57．nfslock：NFS是一个流行的通过TCP/IP网络共享文件的协议，此守护进程提供了NFS文件锁定功能。
58．ntpd：Network Time Protocol Daemon（网络时间校正协议）。ntpd是用来使系统和一个精确的时间源保持时间同步的协议守护进程。
59．network：激活/关闭启动时的各个网络接口守护进程。
60．psacct：该守护进程包括几个工具用来监控进程活动的工具，包括ac，lastcomm，accton和sa。
61．pcmcia：主要用于支持笔记本电脑接口守护进程。
62．portmap：该守护进程用来支持RPC连接，RPC被用于NFS及NIS等服务。
63．postgresql：postgreSQL关系数据库引擎。
64．postfix：postfix是邮件传输代理的守护进程。
65．proftpd：proftpd是UNIX下的一个配置灵活的FTP服务器的守护程序。
66．pppoe：ADSL连接守护进程。
67．random：保存和恢复系统的高质量随机数生成器，这些随机数是系统一些随机行为提供的。
68．rawdevices：在使用集群文件系统时用于加载raw设备的守护进程。
69．readahead、readahead_early：readahead和readahead_early是在Fedora core 2中最新推出的两个后台运行的守护程序。其作用是在启动系统期间，将启动系统所要用到的文件首先读取到内存中，然后在内存中执行，以加快系统的启动速度。
70．rhnsd：Red Hat网络服务守护进程。通知官方的安全信息及为系统打补丁。
71．routed：该守护程序支持RIP协议的自动IP路由表维护。RIP主要使用在小型网络上，大一点的网络就需要复杂一点的协议。
72．rsync：remote sync远程数据备份守护进程。
73．rsh：远程主机上启动一个shell，并执行用户命令。
74．rwhod：允许远程用户获得运行rwho守护程序的机器上所有已登录用户的列表。
75．rstatd：一个为LAN上的其他机器收集和提供系统信息的守候进程。
76．ruserd：远程用户定位服务，这是一个基于RPC的服务，它提供关于当前记录到LAN上一个机器日志中的用户信息
77．rwalld：激活rpc.rwall服务进程，这是一项基于RPC的服务，允许用户给每个注册到LAN机器上的其他终端写消息。
78．rwhod：激活rwhod服务进程，它支持LAN的rwho和ruptime服务。
79．saslauthd：使用SASL的认证守护进程。
80．sendmail：邮件服务器sendmail守护进程。
81．smb：Samba文件共享/打印服务守护进程。
82．snmpd：本地简单网络管理守护进程。
83．squid：代理服务器squid守护进程。
84．sshd：OpenSSH服务器守护进程。Secure Shell Protocol可以实现安全地远程管理主机。
85．smartd：Self Monitor Analysis and Reporting Technology System，监控你的硬盘是否出现故障。
86．syslog：一个让系统引导时启动syslog和klogd系统日志守候进程的脚本。
87．time：该守护进程从远程主机获取时间和日期，采用TCP协议。
88．time-udp：该守护进程从远程主机获取时间和日期，采用UDP协议。
89．tux：在Linux内核中运行Apache服务器的守护进程。
90．vsftpd：vsftpd服务器的守护进程。
91．vncserver：VNC（Virtual Network Computing，虚拟网络计算），它提供了一种在本地系统上显示远程计算机整个"桌面"的轻量型协议。
92．vtun：VPN守护进程。
93．xfs：X Window字型服务器守护进程，为本地和远程X服务器提供字型集。
94．xinetd：支持多种网络服务的核心守护进程。
95．ypbind：为NIS（网络信息系统）客户机激活ypbind服务进程。
96．yppasswdd：NIS密码服务器守护进程。
97．ypserv：NIS主服务器守护进程。
98．yum：RPM操作系统自动升级和软件包管理守护进程。
99．ypxfrd：更快地传输NIS地图。
100．zebra：路由管理守护进程。

---
