# Linux的通用知识

---

## Linux的启动

1、BIOS加载硬盘MBR中的bootloader后，启动过程就被bootloader（GRUB）接管  
2、GRUB加载自己。由于MBR里面空间很小，GRUB2只能放部分代码到里面，所以它采用了好几级的结构来加载自己  
3、bootloader加载内核和initrd image，并启动内核。  
4、内核接管整个系统后，加载/sbin/init（systemd）并创建第一个用户态的进程  
5、init进程开始调用一系列的脚本来创建很多子进程，这些子进程负责初始化整个系统

---

## Linux的多语言支持

Linux支持多种语言，默认的字符编码为en\_US.UTF-8，可以通过localectl命令查看

![](/linux/1.png)

对中文支持可以如下操作：

![](/linux/2.png)

---

## Linux的命令行操作快捷键

![](/linux/3.png)  
可以通过man bash查看

---

## Linux中的帮助命令

man

![](/linux/4.png)

info

![](/linux/5.png)

---

## Linux mount

mount命令的标准格式如下：  
mount -t type -o options device dir  
device: 要挂载的设备（必填）  
dir: 挂载到哪个目录（必填）  
type： 文件系统类型（可选）。大部分情况下都不用指定该参数，系统都会自动检测到设备上的文件系统类型

options： 挂载参数（可选）。

options一般分为两类，一类是Linux VFS所提供的通用参数，就是每个文件系统都可以使用这类参数。另一类是每个文件系统自己支持的特有参数，这个需要参考每个文件系统的文档。

---

## 挂载虚拟文件系统

proc、tmpfs、sysfs、devpts等都是Linux内核映射到用户空间的虚拟文件系统，他们不和具体的物理设备关联，但他们具有普通文件系统的特征，应用层程序可以像访问普通文件系统一样来访问他们。  
mount -t proc none /mnt  
由于proc是内核虚拟的一个文件系统，并没有对应的设备， \#所以这里-t参数必须要指定，这里none可以是任意字符串，因为用mount命令查看挂载点信息时第一列显示的就是这个字符串  
挂载多个设备到一个文件夹在Linux下是支持的，默认会用后面的mount覆盖掉前面的mount，只有当umount后面的device后，原来的device才看的到

---

## Mount bind

bind mount功能非常强大，可以将任何一个挂载点、普通目录或者文件挂载到其他地方，可以在bind的时候指定readonly，这样原来的目录还是能读写，但目的目录为只读  
mkdir -p bind/bind1/sub1  
mkdir -p bind/bind2/sub2  
mount -o bind,ro ./bind/bind1/ ./bind/bind2  
touch ./bind/bind1/sub1/aaa       成功  
touch ./bind/bind2/sub1/aaa       失败

---

## CGroup

CGroup 是 Control Groups 的缩写，是 Linux 内核提供的一种可以限制、记录、隔离进程组 所使用的物理资源 的机制。  
运行一个占用 CPU 的 Java 程序，如果不用 CGroup 物理隔离 CPU 核，那程序会由操作系统层级自动挑选 CPU 核来运行程序。由于操作系统层面采用的是时间片轮询方式随机挑选 CPU 核作为运行容器，所以会在本机器上 24 个 CPU 核上随机执行。如果采用 CGroup 进行物理隔离，我们可以选择某些 CPU 核作为指定运行载体。

### cgroup主要包括下面两部分：

subsystem 一个subsystem就是一个内核模块，它被关联到一颗cgroup树之后，就会在树的每个节点（进程组）上做具体的操作。  
hierarchy 一个hierarchy可以理解为一棵cgroup树，树的每个节点就是一个进程组，每棵树都会与零到多个subsystem关联。

Cpu ：当CPUs比较忙时，用来限制cgroup的CPU使用率；当CPUs不忙时，不产生任何作用。  
Cpuacct：统计cgroup的CPU的使用率。  
Cpuset：绑定cgroup到指定CPUs和NUMA节点。  
memory ：统计和限制cgroup的内存的使用率，包括process memory, kernel memory, 和swap。  
Devices ：限制cgroup创建\(mknod\)和访问设备的权限。  
Freezer：suspend和restore一个cgroup中的所有进程。  
net\_cls：将一个cgroup中进程创建的所有网络包加上一个classid标记，用于iptables。 只对发出去的网络包生效。  
Blkio：限制cgroup访问块设备的IO速度。  
perf\_event ：对cgroup进行性能监控  
net\_prio ：针对每个网络接口设置cgroup的访问优先级。  
Hugetlb：限制cgroup的huge pages的使用量。  
pids ：限制一个cgroup及其子孙cgroup中的总进程数。

cgroup相关的所有操作都是基于内核中的cgroup virtual filesystem  
使用cgroup很简单，挂载这个文件系统就可以了。  
一般情况下都是挂载到/sys/fs/cgroup目录下

mount\|grep cgroup  
mount -t cgroup -o cpu,cpuacct xxx /sys/fs/cgroup/cpu,cpuacct

cgroup.clone\_children：对cpuset（subsystem）有影响，当该文件的内容为1时，新创建的cgroup将会继承父cgroup的配置，即从父cgroup里面拷贝配置文件来初始化新cgroup  
cgroup.procs：当前cgroup中的所有进程ID，系统不保证ID是顺序排列的，且ID有可能重复  
notify\_on\_release：该文件的内容为1时，当cgroup退出时（不再包含任何进程和子cgroup），将调用release\_agent里面配置的命令。新cgroup被创建时将默认继承父cgroup的这项配置。  
release\_agen：t里面包含了cgroup退出时将会执行的命令，系统调用该命令时会将相应cgroup的相对路径当作参数传进去。  
Tasks：当前cgroup中的所有线程ID，系统不保证ID是顺序排列的

