

## 系统中出现大量不可中断进程和僵尸进程

- 当 iowait 升高时，进程很可能因为得不到硬件的响应，而长时间处于不可中断状态。

- R、D、Z、S、I 等几个状态

  - R 是 Running 或 Runnable 的缩写，表示进程在 CPU 的就绪队列中，正在运行或者正在等待运行
  - D 是 Disk Sleep 的缩写，也就是不可中断状态睡眠（Uninterruptible Sleep），一般表示进程正在跟硬件交互，并且交互过程不允许被其他进程或中断打断
  - Z 是 Zombie 的缩写。它表示僵尸进程，也就是进程实际上已经结束了，但是父进程还没有回收它的资源（比如进程的描述符、PID 等）。
    - 这是多进程应用很容易碰到的问题。正常情况下，当一个进程创建了子进程后，它应该通过系统调用 wait() 或者 waitpid() 等待子进程结束，回收子进程的资源；而子进程在结束时，会向它的父进程发送 SIGCHLD 信号，所以，父进程还可以注册SIGCHLD 信号的处理函数，异步回收资源。
    - 如果父进程没这么做，或是子进程执行太快，父进程还没来得及处理子进程状态，子进程就已经提前退出，那这时的子进程就会变成僵尸进程。
  - S 是 Interruptible Sleep 的缩写，也就是可中断状态睡眠，表示进程因为等待某个事件而被系统挂起。当进程等待的事件发生时，它会被唤醒并进入 R 状态。
  - I 是 Idle 的缩写，也就是空闲状态，用在不可中断睡眠的内核线程上。
    - 硬件交互导致的不可中断进程用 D 表示，但对某些内核线程来说，它们有可能实际上并没有任何负载，用 Idle 正是为了区分这种情况。

- iowait 分析

  - dstat ，可以同时查看 CPU 和 I/O 这两种资源的使用情况，便于对比分析。
  - 进程想要访问磁盘，就必须使用系统调用，所以接下来，重点就是找出 app 进程的系统调用了。
    - strace 正是最常用的跟踪进程系统调用的工具
      - 这儿出现了一个奇怪的错误，strace 命令居然失败了，并且命令报出的错误是没有权限
      - 因为进程已经变成了 Z 状态，也就是僵尸进程。
    - 你可以用 perf top 看看有没有新发现
      - 罪魁祸首是进程内部进行了磁盘的直接 I/O

- 僵尸进程

  - 找出父进程，然后在父进程里解决

    - > 1 # -a 表示输出命令行选项
      > 2 # p 表示 PID
      > 3 # s 表示指定进程的父进程
      > 4 $ pstree -aps 3084

-  iowait 高不一定代表 I/O 有性能瓶颈。当系统中只有 I/O 类型的进程在运行时，iowait 也会很高，但实际上，磁盘的读写远没有达到性能瓶颈的程度。

  - 因此，碰到 iowait 升高时，需要先用 dstat、pidstat 等工具，确认是不是磁盘 I/O 的问题，然后再找是哪些进程导致了 I/O。
  - 等待 I/O 的进程一般是不可中断状态，所以用 ps 命令找到的 D 状态（即不可中断状态）的进程，多为可疑进程。
  - 这种情况下，我们用了 perf 工具，来分析系统的 CPU 时钟事件，最终发现是直接 I/O 导致的问题。这时，再检查源码中对应位置的问题，就很轻松了。





## Linux软中断

- 进程的不可中断状态是系统的一种保护机制，可以保证硬件的交互过程不被意外打断。所以，短时间的不可中断状态是很正常的。

- 软中断

  - 为了解决中断处理程序执行过长和中断丢失的问题，Linux 将中断处理过程分成了两个阶段，也就是上半部和下半部
  - 网卡接收数据包的例子
    - 网卡接收到数据包后，会通过硬件中断的方式，通知内核有新的数据到了。这时，内核就应该调用中断处理程序来响应它。
    - 上半部直接处理硬件请求，也就是我们常说的硬中断，特点是快速执行；
    - 而下半部则是由内核触发，也就是我们常说的软中断，特点是延迟执行。
  - 实际上，上半部会打断 CPU 正在执行的任务，然后立即执行中断处理程序。而下半部以内核线程的方式执行，并且每个 CPU 都对应一个软中断内核线程，名字为 “ksoftirqd/CPU编号”，比如说， 0 号 CPU 对应的软中断内核线程的名字就是 ksoftirqd/0。

- 查看软中断和内核线程

  - Linux 中的软中断包括网络收发、定时、调度、RCU 锁等各种类型，可以通过查看/proc/softirqs 来观察软中断的运行情况。

  - > /proc/softirqs 提供了软中断的运行情况；
    > /proc/interrupts 提供了硬中断的运行情况。

  - 每个 CPU 都对应一个软中断内核线程，这个软中断内核线程就叫做 ksoftirqd/CPU 编号

    - > $ ps aux | grep softirq
      > 这些线程的名字外面都有中括号，这说明 ps 无法获取它们的命令行参数。一般来说，ps 的输出中，名字括在中括号里的，一般都是内核线程。





## 系统的软中断CPU使用率升高

- sar、 hping3 和 tcpdump

  - sar 是一个系统活动报告工具，既可以实时查看系统的当前活动，又可以配置保存和报告历史统计数据

    - sar 可以用来查看系统的网络收发情况，还有一个好处是，不仅可以观察网络收发的吞吐量（BPS，每秒收发的字节数），还可以观察网络收发的 PPS，即每秒收发的网络帧数。

    - > 1 # -n DEV 表示显示网络收发的报告，间隔 1 秒输出一组数据
      > 2 $ sar -n DEV 1

  - hping3 是一个可以构造 TCP/IP 协议数据包的工具，可以对系统进行安全审计、防火墙测试等。

    - 运行 hping3 命令，来模拟 Nginx 的客户端请求

    - > 1 # -S 参数表示设置 TCP 协议的 SYN（同步序列号），-p 表示目的端口为 80
      > 2 # -i u100 表示每隔 100 微秒发送一个网络帧
      > 3 # 注：如果你在实践过程中现象不明显，可以尝试把 100 调小，比如调成 10 甚至 1
      > 4 $ hping3 -S -p 80 -i u100 192.168.0.30

  - tcpdump 是一个常用的网络抓包工具，常用来分析各种网络问题。

    - > 1 # -i eth0 只抓取 eth0 网卡，-n 不解析协议名和主机名
      > 2 # tcp port 80 表示只抓取 tcp 协议并且端口号为 80 的网络帧
      > 3 $ tcpdump -i eth0 -n tcp port 80

- $ watch -d cat /proc/softirqs

  - TIMER（定时中断）、NET_RX（网络接收）、NET_TX（网络发送）、SCHED（内核调度）、RCU（RCU 锁）等这几个软中断都在不停变化
  - NET_RX，也就是网络数据包接收软中断的变化速率最快。
  - 而其他几种类型的软中断，是保证 Linux 调度、时钟和临界区保护这些正常工作所必需的，所以它们有一定的变化倒是正常的。

- 到这里，我们已经做了全套的性能诊断和分析。从系统的软中断使用率高这个现象出发，通过观察 /proc/softirqs 文件的变化情况，判断出软中断类型是网络接收中断；再通过 sar和 tcpdump ，确认这是一个 SYN FLOOD 问题。

  - SYN FLOOD 问题最简单的解决方法，就是从交换机或者硬件防火墙中封掉来源 IP，这样SYN FLOOD 网络帧就不会发送到服务器中。





## 如何迅速分析出系统CPU的瓶颈在哪里

- CPU 性能指标
  - 最容易想到的应该是 CPU 使用率
    - CPU 使用率描述了非空闲时间占总 CPU 时间的百分比，根据 CPU 上运行任务的不同，又被分为用户 CPU、系统 CPU、等待 I/O CPU、软中断和硬中断等。
  - 平均负载（Load Average）
  - 进程上下文切换
    - 无法获取资源而导致的自愿上下文切换；
    - 被系统强制调度导致的非自愿上下文切换。
  - 还有一个指标，CPU 缓存的命中率

- 性能工具
  - 平均负载的案例。我们先用 uptime， 查看了系统的平均负载；而在平均负载升高后，又用 mpstat 和 pidstat ，分别观察了每个 CPU 和每个进程 CPU 的使用情况，进而找出了导致平均负载升高的进程，也就是我们的压测工具stress。
  - 第二个，上下文切换的案例。我们先用 vmstat ，查看了系统的上下文切换次数和中断次数；然后通过 pidstat ，观察了进程的自愿上下文切换和非自愿上下文切换情况；最后通过pidstat ，观察了线程的上下文切换情况，找出了上下文切换次数增多的根源，也就是我们的基准测试工具 sysbench。
  - 第三个，进程 CPU 使用率升高的案例。我们先用 top ，查看了系统和进程的 CPU 使用情况，发现 CPU 使用率升高的进程是 php-fpm；再用 perf top ，观察 php-fpm 的调用链，最终找出 CPU 升高的根源，也就是库函数 sqrt() 。
  - 第四个，系统的 CPU 使用率升高的案例。我们先用 top 观察到了系统 CPU 升高，但通过top 和 pidstat ，却找不出高 CPU 使用率的进程。于是，我们重新审视 top 的输出，又从CPU 使用率不高但处于 Running 状态的进程入手，找出了可疑之处，最终通过 perfrecord 和 perf report ，发现原来是短时进程在捣鬼。
  - 第五个，不可中断进程和僵尸进程的案例。我们先用 top 观察到了 iowait 升高的问题，并发现了大量的不可中断进程和僵尸进程；接着我们用 dstat 发现是这是由磁盘读导致的，于是又通过 pidstat 找出了相关的进程。但我们用 strace 查看进程系统调用却失败了，最终还是用 perf 分析进程调用链，才发现根源在于磁盘直接 I/O 。
  - 最后一个，软中断的案例。我们通过 top 观察到，系统的软中断 CPU 使用率升高；接着查看 /proc/softirqs， 找到了几种变化速率较快的软中断；然后通过 sar 命令，发现是网络小包的问题，最后再用 tcpdump ，找出网络帧的类型和来源，确定是一个 SYN FLOOD攻击导致的。

- **性能指标和性能工具**
  - 第一个维度，从 CPU 的性能指标出发。也就是说，当你要查看某个性能指标时，要清楚知道哪些工具可以做到。
    - ![img](https://upload-images.jianshu.io/upload_images/11462765-6d4f67c35d53bf18.png?imageMogr2/auto-orient/strip|imageView2/2/w/640/format/webp)
  - 第二个维度，从工具出发。也就是当你已经安装了某个工具后，要知道这个工具能提供哪些指标。
    - ![img](https://upload-images.jianshu.io/upload_images/11462765-7d6c6ae539f628cc.png?imageMogr2/auto-orient/strip|imageView2/2/w/640/format/webp)
  - 为了缩小排查范围，我通常会先运行几个支持指标较多的工具，如 top、vmstat 和pidstat
    - ![img](https://upload-images.jianshu.io/upload_images/11462765-faed248242889ff0.png?imageMogr2/auto-orient/strip|imageView2/2/w/640/format/webp)
  - **第一个例子，pidstat 输出的进程用户 CPU 使用率升高**，会导致 top 输出的用户 CPU 使用率升高。所以，当发现 top 输出的用户 CPU 使用率有问题时，可以跟 pidstat 的输出做对比，观察是否是某个进程导致的问题。
    - 而找出导致性能问题的进程后，就要用进程分析工具来分析进程的行为，比如使用 strace 分析系统调用情况，以及使用 perf 分析调用链中各级函数的执行情况。
  - **第二个例子，top 输出的平均负载升高**，可以跟 vmstat 输出的运行状态和不可中断状态的进程数做对比，观察是哪种进程导致的负载升高。
  - **最后一个例子，当发现 top 输出的软中断 CPU 使用率升高时**，可以查看 /proc/softirqs 文件中各种类型软中断的变化情况，确定到底是哪种软中断出的问题。比如，发现是网络接收中断导致的问题，那就可以继续用网络分析工具 sar 和 tcpdump 来分析。



## **CPU** **性能优化的几个思路**

- **怎么评估性能优化的效果**
  - 应用程序的维度，我们可以用**吞吐量和请求延迟**来评估应用程序的性能。 
  - 系统资源的维度，我们可以用 **CPU 使用率**来评估系统的 CPU 使用情况。 
  - 还是以刚刚的 Web 应用为例，对应上面提到的几个指标，我们可以选择 ab 等工具，测试Web 应用的并发请求数和响应延迟。而测试的同时，还可以用 vmstat、pidstat 等性能工具，观察系统和进程的 CPU 使用率。这样，我们就同时获得了应用程序和系统资源这两个 维度的指标数值。
- **CPU** **优化**
  - **应用程序优化**
    - 比如减少循环的层次、减少递归、减少动态内存分配等等。
    - **编译器优化**
    - **算法优化**
    - **异步处理**
    - **多线程代替多进程**
    - **善用缓存**
  - **系统优化**
    - 从系统的角度来说，优化 CPU 的运行，一方面要充分利用 CPU 缓存的本地性，加速缓存访问；另一方面，就是要控制进程的 CPU 使用情况，减少进程间的相互影响。 
    - **CPU 绑定**：把进程绑定到一个或者多个 CPU 上，可以提高 CPU 缓存的命中率，减少跨 CPU 调度带来的上下文切换问题。 
    - **CPU 独占**：跟 CPU 绑定类似，进一步将 CPU 分组，并通过 CPU 亲和性机制为其分配 进程。这样，这些 CPU 就由指定的进程独占，换句话说，不允许其他进程再来使用这些 CPU。 
    - **优先级调整**：使用 nice 调整进程的优先级，正值调低优先级，负值调高优先级。优先级 的数值含义前面我们提到过，忘了的话及时复习一下。在这里，适当降低非核心应用的优 先级，增高核心应用的优先级，可以确保核心应用得到优先处理。 
    - **为进程设置资源限制**：使用 Linux cgroups 来设置进程的 CPU 使用上限，可以防止由于 某个应用自身的问题，而耗尽系统资源。 
    - **NUMA（Non-Uniform Memory Access）优化**：支持 NUMA 的处理器会被划分为 多个 node，每个 node 都有自己的本地内存空间。NUMA 优化，其实就是让 CPU 尽可能只访问本地内存。 
    - **中断负载均衡**：无论是软中断还是硬中断，它们的中断处理程序都可能会耗费大量的CPU。开启 irqbalance 服务或者配置 smp_affinity，就可以把中断处理过程自动负载均 衡到多个 CPU 上。