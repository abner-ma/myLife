# Linux下的系统性能调优工具――Perf

## 背景
内存读写是很快的，但还是无法和处理器的指令执行速度相比。为了从内存中读取指令和数据，处理器需要等待，用处理器的时间来衡量，这种等待非常漫长。Cache是一种SRAM，它的读写速率非常快，能和处理器处理速度相匹配。因此将常用的数据保存在cache中，处理器便无须等待，从而提高性能。
Cache的尺寸一般都很小，充分利用Cache是软件调优非常重要的部分。
提高性能最有效的方式之一就是并行。处理器在硬件设计时也尽可能地并行，比如流水线，超标量体系结构以及乱序执行。
处理器处理一条指令需要分多个步骤完成，比如先取指令，然后完成运算，最后将计算结果输出到总线上。
第一条指令在运算时，第二条指令已经在取指了;而第一条指令输出结果时，第二条指令则又可以运算了，而第三天指令也已经取指了。
这样从长期看来，像是三条指令在同时执行。在处理器内部，这可以看作一个三级流水线。
超标量(superscalar)指一个时钟周期发射多条指令的流水线机器架构，比如 Intel 的 Pentium 处理器，内部有两个执行单元，在一个时钟周期内允许执行两条指令。
在处理器内部，不同指令所需要的处理步骤和时钟周期是不同的，如果严格按照程序的执行顺序执行，那么就无法充分利用处理器的流水线。因此指令有可能被乱序执行。
上述三种并行技术对所执行的指令有一个基本要求，即相邻的指令相互没有依赖关系。假如某条指令需要依赖前面一条指令的执行结果数据，那么 pipeline 便失去作用，因为第二条指令必须等待第一条指令完成。因此好的软件必须尽量避免这种代码的生成。

分支指令对软件性能有比较大的影响。尤其是当处理器采用流水线设计之后，假设流水线有三级，当前进入流水的第一条指令为分支指令。假设处理器顺序读取指令，那么如果分支的结果是跳转到其他指令，那么被处理器流水线预取的后续两条指令都将被放弃，从而影响性能。
为此，很多处理器都提供了分支预测功能，根据同一条指令的历史执行记录进行预测，读取最可能的下一条指令，而并非顺序读取指令。
分支预测对软件结构有一些要求，对于重复性的分支指令序列，分支预测硬件能得到较好的预测结果，而对于类似 switch case 一类的程序结构，则往往无法得到理想的预测结果。

上面介绍的几种处理器特性对软件的性能有很大的影响，然而依赖时钟进行定期采样的 profiler 模式无法揭示程序对这些处理器硬件特性的使用情况。
处理器厂商针对这种情况，在硬件中加入了 PMU 单元，即 performance monitor unit。PMU 允许软件针对某种硬件事件设置 counter，此后处理器便开始统计该事件的发生次数，当发生的次数超过 counter 内设置的值后，便产生中断。
比如 cache miss 达到某个值后，PMU 便能产生相应的中断。捕获这些中断，便可以考察程序对这些硬件特性的利用效率了。

---

## Tracepoints

Tracepoint 是散落在内核源代码中的一些 hook，一旦使能，它们便可以在特定的代码被运行到时被触发，这一特性可以被各种 trace/debug 工具所使用。Perf 就是该特性的用户之一。
假如您想知道在应用程序运行期间，内核内存管理模块的行为，便可以利用潜伏在 slab 分配器中的 tracepoint。当内核运行到这些 tracepoint 时，便会通知 perf。
Perf 将 tracepoint 产生的事件记录下来，生成报告，通过分析这些报告，调优人员便可以了解程序运行时期内核的种种细节，对性能症状作出更准确的诊断。

---

## Perf

### Perf简介
Perf是Linux内核自带的系统性能优化工具，原理是：
CPU的PMU registers中Get/Set performance counters来获得诸如instructions executed, cache-missed suffered, branches mispredicted等信息。
linux kernel对这些registers进行了一系列抽象，所以你可以按进程，按CPU或者按counter group等不同类别来查看Sample信息。
通过Perf，应用程序可以利用 PMU，tracepoint 和内核中的特殊计数器来进行性能统计。
它不但可以分析指定应用程序的性能问题 (per thread)，也可以用来分析内核的性能问题，当然也可以同时分析应用代码和内核，从而全面理解应用程序中的性能瓶颈。
Perf 不仅可以用于应用程序的性能统计分析，也可以应用于内核代码的性能统计和分析。得益于其优秀的体系结构设计，越来越多的新功能被加入 Perf，使其已经成为一个多功能的性能统计工具集 。

### Perf基本使用

性能调优工具如 perf，Oprofile 等的基本原理都是对被监测对象进行采样，最简单的情形是根据 tick 中断进行采样，即在 tick 中断内触发采样点，在采样点里判断程序当时的上下文。
假如一个程序 90% 的时间都花费在函数 foo() 上，那么 90% 的采样点都应该落在函数 foo() 的上下文中。
运气不可捉摸，但我想只要采样频率足够高，采样时间足够长，那么以上推论就比较可靠。因此，通过 tick 触发采样，我们便可以了解程序中哪些地方最耗时间，从而重点分析。
稍微扩展一下思路，就可以发现改变采样的触发条件使得我们可以获得不同的统计数据：以时间点(如 tick) 作为事件触发采样便可以获知程序运行时间的分布。
以 cache miss 事件触发采样便可以知道 cache miss 的分布，即 cache 失效经常发生在哪些程序代码中。

使用 perf list 命令可以列出所有能够触发 perf 采样点的事件。
不同的系统会列出不同的结果，在 2.6.35 版本的内核中，该列表已经相当的长，但无论有多少，我们可以将它们划分为三类：
Hardware Event 是由 PMU 硬件产生的事件，比如 cache 命中，当您需要了解程序对硬件特性的使用情况时，便需要对这些事件进行采样; 
Software Event 是内核软件产生的事件，比如进程切换，tick 数等;
Tracepoint event 是内核中的静态 tracepoint 所触发的事件，这些 tracepoint 用来判断程序运行期间内核的行为细节，比如 slab 分配器的分配次数等。
上述每一个事件都可以用于采样，并生成一项统计数据。

面对一个问题程序，最好采用自顶向下的策略。先整体看看该程序运行时各种统计事件的大概，再针对某些方向深入细节。而不要一下子扎进琐碎细节，会一叶障目的。
有些程序慢是因为计算量太大，其多数时间都应该在使用 CPU 进行计算，这叫做 CPU bound 型;
有些程序慢是因为过多的 IO，这种时候其 CPU 利用率应该不高，这叫做 IO bound 型;
对于 CPU bound 程序的调优和 IO bound 的调优是不同的。所以Perf stat 应该是最先使用的一个工具。它通过概括精简的方式提供被调试程序运行的整体情况和汇总数据。

perf stat 给出了其他几个最常用的统计信息：
Task-clock-msecs：CPU 利用率，该值高，说明程序的多数时间花费在 CPU 计算上而非 IO。
Context-switches：进程切换次数，记录了程序运行过程中发生了多少次进程切换，频繁的进程切换是应该避免的。
Cache-misses：程序运行过程中总体的 cache 利用情况，如果该值过高，说明程序的 cache 利用不好。
CPU-migrations：表示进程运行过程中发生了多少次 CPU 迁移，即被调度器从一个 CPU 转移到另外一个 CPU 上运行。
Cycles：处理器时钟，一条机器指令可能需要多个 cycles；
Instructions: 机器指令数目。
IPC：是 Instructions/Cycles 的比值，该值越大越好，说明程序充分利用了处理器的特性。
Cache-references: cache 命中的次数。  
通过指定 -e 选项，您可以改变 perf stat的缺省事件(关于事件，可以通过perf list来查看)。假如您已经有很多的调优经验，可能会使用-e 选项来查看您所感兴趣的特殊的事件。

Perf top 用于实时显示当前系统的性能统计信息。
该命令主要用来观察整个系统当前的状态，比如可以通过查看该命令的输出来查看当前系统最耗时的内核函数或某个用户进程。
perf record时加上-g选项，可以记录函数的调用关系。

当 perf 根据 tick 时间点进行采样后，人们便能够得到内核代码中的 hot spot。使用 tracepoint 的基本需求是对内核的运行时行为的关心，有些内核开发人员需要专注于特定的子系统，比如内存管理模块。这便需要统计相关内核函数的运行情况。

pref probe:tracepoint 是静态检查点，意思是一旦它在哪里，便一直在那里了，您想让它移动一步也是不可能的。
Perf 并不是第一个提供这个功能的软件，systemTap 早就实现了。但假若您不选择 RedHat 的发行版的话，安装 systemTap 并不是件轻松愉快的事情。perf 是内核代码包的一部分，所以使用和维护都非常方便。
利用 probe 命令在内核函数某处加入了一个动态 probe 点，和 tracepoint 的功能一样，内核一旦运行到该 probe 点时，便会通知 perf。
可以理解为动态增加了一个新的 tracepoint。此后便可以用 record 命令的 -e 选项选择该 probe 点，最后用 perf report 查看报表。
Perf sched：调度器的好坏直接影响一个系统的整体运行效率。
在这个领域，内核黑客们常会发生争执，一个重要原因是对于不同的调度器，每个人给出的评测报告都各不相同，甚至常常有相反的结论。因此一个权威的统一的评测工具将对结束这种争论有益。
用户一般使用perf sched record收集调度相关的数据，然后就可以用perf sched latency查看诸如调度延迟等和调度器相关的统计数据。
其他三个命令也同样读取 record 收集到的数据并从其他不同的角度来展示这些数据。
各个 column 的含义如下： Task: 进程的名字和 pid
Runtime: 实际运行时间
Switches: 进程切换的次数
Average delay: 平均的调度延迟
Maximum delay: 最大延迟
这里最值得人们关注的是 Maximum delay，一般从这里可以看到对交互性影响最大的特性：调度延迟，如果调度延迟比较大，那么用户就会感受到视频或者音频断断续续的。
其他的三个子命令提供了不同的视图，一般是由调度器的开发人员或者对调度器内部实现感兴趣的人们所使用。
首先是 map:
星号表示调度事件发生所在的 CPU。
点号表示该 CPU 正在 IDLE。
Map 的好处在于提供了一个的总的视图，将成百上千的调度事件进行总结，显示了系统任务在 CPU 之间的分布，假如有不好的调度迁移，比如一个任务没有被及时迁移到 idle 的 CPU 却被迁移到其他忙碌的 CPU，类似这种调度器的问题可以从 map 的报告中一眼看出来。
如果说 map 提供了高度概括的总体的报告，那么 trace 就提供了最详细，最底层的细节报告。
Perf replay 这个工具更是专门为调度器开发人员所设计，它试图重放 perf.data 文件中所记录的调度场景。
很多情况下，一般用户假如发现调度器的奇怪行为，他们也无法准确说明发生该情形的场景，或者一些测试场景不容易再次重现，或者仅仅是出于“偷懒”的目的，使用 perf replay，perf 将模拟 perf.data 中的场景，无需开发人员花费很多的时间去重现过去，这尤其利于调试过程，因为需要一而再，再而三地重复新的修改是否能改善原始的调度场景所发现的问题。
除了调度器之外，很多时候人们都需要衡量自己的工作对系统性能的影响。benchmark 是衡量性能的标准方法，对于同一个目标，如果能够有一个大家都承认的 benchmark，将非常有助于”提高内核性能”这项工作。
perf bench至少提供了 3 个 benchmark:
1.Sched message
sched message 是从经典的测试程序 hackbench 移植而来，用来衡量调度器的性能，overhead 以及可扩展性。该 benchmark 启动 N 个 reader/sender 进程或线程对，通过 IPC(socket 或者 pipe) 进行并发的读写。一般人们将 N 不断加大来衡量调度器的可扩展性。Sched message 的用法及用途和 hackbench 一样。
2.Sched Pipe
Sched pipe 从 Ingo Molnar 的 pipe-test-1m.c 移植而来。当初 Ingo 的原始程序是为了测试不同的调度器的性能和公平性的。
其工作原理很简单，两个进程互相通过 pipe 拼命地发 1000000 个整数，进程 A 发给 B，同时 B 发给 A。。。因为 A 和 B 互相依赖，因此假如调度器不公平，对 A 比 B 好，那么 A 和 B 整体所需要的时间就会更长。
3.Mem memcpy
这个是 perf bench 的作者 Hitoshi Mitake 自己写的一个执行 memcpy 的 benchmark。该测试衡量一个拷贝 1M 数据的 memcpy() 函数所花费的时间。
perf lock
锁是内核同步的方法，一旦加了锁，其他准备加锁的内核执行路径就必须等待，降低了并行。因此对于锁进行专门分析应该是调优的一项重要工作。
perf Kmem
Perf Kmem 专门收集内核 slab 分配器的相关事件。比如内存分配，释放等。可以用来研究程序在哪里分配了大量内存，或者在什么地方产生碎片之类的和内存管理相关的问题。
Perf kmem 和 perf lock 实际上都是 perf tracepoint 的特例，您也完全可以用 Perf record – e kmem: 或者 perf record – e lock: 来完成同样的功能。但重要的是，这些工具在内部对原始数据进行了汇总和分析，因而能够产生信息更加明确更加有用的统计报表。
Perf timechart：很多 perf 命令都是为调试单个程序或者单个目的而设计。有些时候，性能问题并非由单个原因所引起，需要从各个角度一一查看。为此，人们常需要综合利用各种工具，比如 top,vmstat,oprofile 或者 perf。这非常麻烦。
此外，前面介绍的所有工具都是基于命令行的，报告不够直观。更令人气馁的是，一些报告中的参数令人费解。
所以人们更愿意拥有一个“傻瓜式”的工具。
以上种种就是 perf timechart 的梦想，其灵感来源于 bootchart。采用“简单”的图形“一目了然”地揭示问题所在。
Timechart 可以显示更详细的信息，实际上是一个矢量图形 SVG 格式，用 SVG viewer 的放大功能，我们可以将该图的细节部分放大，timechart 的设计理念叫做”infinitely zoomable”。放大之后便可以看到一些更详细的信息，类似网上的 google 地图，找到国家之后，可以放大，看城市的分布，再放大，可以看到某个城市的街道分布，还可以放大以便得到更加详细的信息。