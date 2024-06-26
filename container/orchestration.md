# 7.2 从 Container 到 Pod

字面上，“Container”这个词很难让人形象地理解其真正含义，Kubernetes 中的“Pod”也是如此。

仅凭几句解释并不能充分认识这些概念，甚至还会引起误解，譬如把容器与轻量化虚拟机混为一谈。

:::tip 如果容器类似虚拟机
那就应该有一种普适的方法把虚拟机里面的应用无缝地迁移到容器中，可是现实世界中并没有这种方法。
:::

为了尽可能深入理解容器，我们从最原始的文件系统隔离讲起。

## 7.2.1 文件系统隔离

1979年，Unix 系统引入了一个革命性的技术 —— chroot（chang root）。

chroot 允许管理员将进程的根目录锁定在指定的位置，从而限制该进程访问的文件系统范围。chroot 的隔离能力对安全性至关重要，比如创建一个隔离环境，用来安全地运行和监控可疑的代码或者程序，因此 chroot 之后的环境也被形象地称为jail（监狱）。

仅需几步，就能创建一个 chroot 环境。

```bash
$ mkdir -p new-root/{bin,lib64,root}
$ cp /bin/bash new-root/bin
$ cp /lib64/{ld-linux-x86-64.so*,libc.so*,libdl.so.2,libreadline.so*,libtinfo.so*} new-root/lib64
$ sudo chroot new-root
```
这个 jail 用处不大，只有 bash 以及一些内置的函数，但也足以说明它的作用“运行在此 jail 下进程的文件系统与宿主机隔离了”。

```bash
bash-4.2# cd bin 
bash-4.2# pwd
/bin
```
:::tip 额外知识

除了 /bin 之外，如果我们把程序依赖的 /etc，/proc 等等都打包进去，实际上就得到了一个 rootfs 文件。**由于 rootfs 里打包的不只是应用，而是整个操作系统的文件和目录，也就意味着，应用以及它运行所需要的所有依赖，都被封装在了一起**，这就是容器被反复宣传至今一致性的由来。
:::

我们再运行一个 docker，看看两者之间的区别。

```bash
$ docker run -t -i ubuntu:18.04 /bin/bash

root@028f46a5b7db:/# cd bin
root@028f46a5b7db:/bin# pwd
/bin
```
看起来跟 chroot 差不多，也是一个与宿主机隔离的文件系统环境，那这是否意味着 chroot 就是容器了呢？

肯定不是，**chroot 只是改变了根目录，而非创建了真正的独立隔离、安全的环境**，chroot 内的进程通过几行代码就能从当前的 jail 中逃逸，而且文件系统、网络、设备等等都没有被充分隔离。

## 7.2.2 资源全方位隔离

chroot 最初的目的是为了实现文件的隔离，并非为了容器而设计。

后来 Linux 吸收了 chroot 的理念，先是在 2.4.19 引入了 Mount 命名空间，这样就可以隔离挂载文件系统。又想到进程间通信也需要隔离，就有了 IPC 命名空间。同时，容器还需要一个独立的主机名以便在网络中标识自己，有了网络，自然还要有独立的 IP、端口、路由等...。

从 Linux 内核 2.6.19 起，陆陆续续添加了 UTS、IPC、PID、Network、User 等命名空间。至 Linux 内核 3.8 版本，Linux 已经完成容器所需的 6 项最基本资源隔离。

:::center
表 7-1 Linux 目前支持的八类命名空间
:::

| 命名空间 | 隔离内容 | 内核版本|
|:--|:--|:--|
| Mount| 隔离文件系统挂载点 | 2.4.19 |
| IPC| 隔离进程间通信，使进程拥有独立消息队列、共享内存和信号量 | 2.6.19|
| UTS| 隔离主机的 Hostname、Domain names，这样容器就可以拥有独立的主机名和域名，在网络中可以被视作一个独立的节点。 | 2.6.19 |
| PID| 隔离进程号，对进程 PID 重新编码，不同命名空间下的进程可以有相同的 PID | 2.6.24 |
| Network| 隔离网络资源，包括网络设备、协议栈（IPv4、IPv6）、IP 路由表、iptables、套接字（socket）等 | 2.6.29 |
| User| 隔离用户和用户组 | 3.8 |
| Cgroup| 使进程拥有一个独立的 cgroup 控制组 | 4.6 |
| Time| 隔离系统时间 | 5.6 |


我们创建子进程通常使用 fork()，fork 背后调用的是 clone()，如果要为创建的子进程设置各类资源隔离，使用 clone 并指定 flags 参数即可。

```c
int clone(int (*fn)(void *), void *child_stack,
         int flags, void *arg, ...
         /* pid_t *ptid, struct user_desc *tls, pid_t *ctid */ );
```

如下代码所示，通过 clone 创建子进程，新建的子进程将会“看到”一个全新的系统环境。这个环境内，挂载的文件目录、进程 PID、进程通信资源、网络及网络设备、UTS 等，全部与宿主机隔离。

```c
int flags = CLONE_NEWNS | CLONE_NEWPID | CLONE_NEWIPC | CLONE_NEWNET | CLONE_NEWUTS;
int pid = clone(main_function, stack_size, flags | SIGCHLD, NULL); 
```

## 7.2.3 资源全方位限制

进程的资源隔离已经完成，如果再对使用资源进行额度限制，就能对进程的运行环境实现一个进乎完美的隔离。

这就要用 Linux 内核的第二项技术： Linux Control Cgroup —— 简称 cgroups。

:::tip cgroups（控制群组）

cgroups 是一种内核级别的资源管理机制，可以实现对 Linux 进程或者进程组的资源限制、隔离和统计功能，最早由 Google 工程师 Paul Menage 和 Rohit Seth 于 2006 年发起，在 2008年合并到 2.6.24 版内核后正式对外发布，这一阶段的 cgroups 被称为第一代 cgroups。

2016 年 3 月发布的 Linux 内核 4.5 版本中，搭载了由 Facebook 工程师 Tejun Heo 重新编写的“第二代 cgroups”，相较于 v1 版本，Facebook 工程师编写的 cgroups 提供了更加统一的资源控制接口，使得对于 CPU、内存、I/O 等资源的限制更加一致和统一。

不过由于兼容性和稳定性原因，目前多数容器运行时默认使用的是第一代 cgroups。
:::


Linux 系统执行 ` ll /sys/fs/cgroup `，可以看到有很多 blkio、cpu 这样的目录。这也叫子系统，子系统显示了当前机器可被 cgroups 限制的资源种类。

```bash
$ ll /sys/fs/cgroup
总用量 0
drwxr-xr-x 2 root root  0 2月  17 2023 blkio
lrwxrwxrwx 1 root root 11 2月  17 2023 cpu -> cpu,cpuacct
lrwxrwxrwx 1 root root 11 2月  17 2023 cpuacct -> cpu,cpuacct
...
```
在对应的子系统内，创建目录，比如。

```bash
$ mkdir /sys/fs/cgroup/memory/test
$ ls /sys/fs/cgroup/memory/test
cgroup.clone_children               memory.memsw.failcnt
cgroup.event_control                memory.memsw.limit_in_bytes
```
这样的目录被称为”控制群组“，控制群组创建后会自动生成一系列资源限制文件。目前，Linux 系统一般支持如下控制群组。

:::center
表 7-2 cgroups 控制群组子系统
:::
| 控制群组子系统 | 功能|
|:--|:--|
|blkio | 控制并监控 cgroup 中的任务对块设备(例如磁盘、USB 等) I/O 的存取 |
| cpu | 控制 cgroups 中进程的 CPU 占用率 |
|cpuacct| 自动生成报告来显示 cgroup 中的进程所使用的 CPU 资源 |
| cpuset| 可以为 cgroups 中的进程分配独立 CPU 和内存节点 |
|devices | 控制 cgroups 中进程对某个设备的访问权限|
|freezer | 暂停或者恢复 cgroup 中的任务 |
| memory | 自动生成 cgroup 任务使用内存资源的报告，并限定这些任务所用内存的大小 |
|net_cls | 使用等级识别符（classid）标记网络数据包，这让 Linux 流量管控器（tc）可以识别从特定 cgroup 中生成的数据包 ，可配置流量管控器，让其为不同 cgroup 中的数据包设定不同的优先级|
| net_prio | 可以为各个 cgroup 中的应用程序动态配置每个网络接口的流量优先级 |
|perf_event | 允许使用 perf 工具对 crgoups 中的进程和线程监控|


控制群组内的资源限制往文件内写入资源管理配置，最后则是将进程 PID 写入 tasks 文件里，然后配置生效。

如下所示，3892 这个进程内存被限制在 1 GB、只允许使用 1/4 CPU 时间。 

```bash
/sys/fs/cgroup/memory/$hostname/memory.limit_in_bytes=1GB // 容器进程及其子进程使用的总内存不超过 1GB
/sys/fs/cgroup/cpu/$hostname/cpu.shares=256 // CPU 总 slice 是 1024，因此限制进程最多只能占用 1/4 CPU 时间

echo 3892 > /sys/fs/cgroup/cpu/$hostname/tasks 
```

至此，相信读者们也一定理解容器是什么，容器不是轻量化的虚拟机，也没有创造出真正的沙盒（容器之间共享系统内核，这也是为什么又出现了 kata、gVisor 等内核隔离的沙盒容器），只是使用了 Namespace、cgroups 等技术进行资源隔离、限制以及拥有独立 rootfs 的特殊进程。

## 7.2.4 设计容器协作的方式

既然容器是个特殊的进程，那联想到真正的操作系统内大部分的进程也并非独自运行，而是以进程组有原则的组织在一起，共同协作完成某项工作。

登录到一台 Linux 机器，执行 pstree -g 命令展示当前系统中正在运行的进程树状结构。

```bash
$ pstree -g
    |-rsyslogd(1089)-+-{in:imklog}(1089)
    |  |-{in:imuxsock) S 1(1089)
    | `-{rs:main Q:Reg}(1089)
```
如上输出，展示了 Linux 系统中负责处理日志的 rsyslogd 程序的进程树结构。可见 rsyslogd 的主程序 main 以及它要用到的内核日志模块 imklog 等同属 1089 进程组，这些进程相互协作，共享 rsyslogd 程序的资源，共同完成 rsyslogd 程序的职责。

对于操作系统而言，这种进程组也更方便管理，Linux 操作系统只需将信号（如 SIGKILL）发给一个进程组，该进程组中的所有进程就都会收到这个信号而终止运行。

那么，现在思考一个问题，如果把上面的进程用容器改造跑起来，该如何设计？

如果是使用 Docker，自然会想到在 Docker 容器内运行两个进程：
- rsyslogd 执行业务；
- imklog 处理业务日志。

但这样设计会有一个问题：**容器里面 PID=1 的进程该是谁**？这个问题的核心在于 **Docker 容器的设计本身是一种“单进程”模型**，Docker 只能通过监视 PID 为 1 的进程的运行状态来判断容器的工作状态是否正常（即由 ENTRYPOINT 启动的进程）。

:::tip 额外知识
在 Linux 系统中，PID 为 1 的进程是 init，它是所有其他进程的祖先进程。它不断地检查进程状态，一旦某个子进程因为父进程错误提前退出而成为孤儿进程，init 就会收养这个子进程。所以，实现的容器中，启动的第一个进程也需要实现类似 init 功能，处理 SIGTERM\SIGINT 等信号，优雅结束容器内的进程。

因此，如果确实需要在一个 Docker 容器中运行多个进程，最先启动的命令进程应该具有资源监控和回收等管理能力，譬如使用针对容器开发的 tinit 程序。
:::

如果容器想要实现类似操作系统进程组那般互相协作，容器下一步的演进就是要找到与“进程组”相对应的概念，这是实现容器从隔离到协作的第一步。

## 7.2.5 超亲密容器组 Pod

Kubernetes 中这个设计叫做 Pod，Pod 是一组紧密关联的容器集合，它们共享 IPC、Network、UTS 等命名空间，是 Kubernetes 最基本单位。

容器之间原本是被 Linux Namespace 和 cgroups 隔开的，Pod 第一个要解决的问题是怎么去打破这个隔离，让 Pod 内的容器可以像进程组一样天然的共享资源和数据。

Kubernetes 使用了一个特殊的容器（Infra Container）解决这个了问题。

Infra Container 是整个 Pod 中第一个启动的容器，只有 300 KB 左右大小，它负责申请容器组的 UTS、IPC、网络等命名空间，Pod 内其他容器通过 setns（Linux 系统调用，把进程加入到某个命名空间中）方式共享 Infra Container 容器的命名空间，其次它还可作为 init 进程，用来管理子进程、回收资源等。

:::tip 额外知识
Infra Container 中的代码仅是注册 SIGTERM、SIGINT、SIGCHILD 等信号处理，启动之后执行一个永远循环的 pause() 方法，所以也常被称为“pause 容器”。
:::

:::center
  ![](../assets/infra-container.svg) <br/>
  图 7-4 Pod 内的容器通过 Infra Container 共享网络命名空间
:::

通过 Infra Container，同一 Pod 内的容器共享 UTS、Network、IPC、Time 命名空间。

注意，PID 命名空间和文件命名空间默认还是隔离的，这是因为：
- 容器之间也需要相互独立的文件系统以避免冲突。如果容器之间想要想要实现文件共享，Kubernetes 也提供了 Volume 支持（Volume 的概念将在本章 7.5 节介绍）。
- PID 隔离是因为如果某些容器进程不再具备 PID=1，容器可能会拒绝启动（例如使用 systemd 的容器）。

如果要共享 PID 命名空间，需要设置 PodSpec 中的 ShareProcessNamespace 为 true，如下 yaml 所示。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  shareProcessNamespace: true
  ...
```

设置之后，Infra Container 将作为 PID 1 进程，由 Infra Container 负责信号处理、子进程的资源回收等。

## 7.2.6 Pod 是 Kubernetes 的基本单位

解决了容器的协作问题，围绕容器和 Pod 不断向实际应用的场景扩展。

因为 Pod 不可能只有一个实例，于是有了 Deployment，实现 Pod 多个实例运行。因为 Pod 是动态变化的额，还得有一个唯一的访问入口，并在多个实例之间负载均衡，于是就有了 Service。Service 是基于四层 TCP 和 UDP 协议转发，还得有应用层协议 (HTTP/HTTPS）访问支持，并通过域名/路径做到更细粒度的划分，于是就有了 Ingress。

围绕 Pod，最终绘制出如图 7-5 所示 Kubernetes 核心功能全景图。

:::center
  ![](../assets/pod.svg)<br/>
  图 7-5 Kubernetes 核心功能全景图
:::

## 7.2.7 Pod 是调度的原子单位

Pod 承担的另外一个重要职责是 —— 作为调度的原子单位。

协同调度是非常麻烦的事情。举个例子说明，有以下两个亲和性容器：
- 第一个容器 Nginx（资源需求 1G 内存） 接收请求，并将请求写入日志文件。
- 第二个容器 LogCollector（资源需求 0.5 G 内存），它会把 Nginx 容器写的日志文件转发到后端的 ElasticSearch 中。

当前集群环境的可用资源是这样一个情况：Node1 1.25G 内存，Node2 2G 内存。

假设这两个 Pod 因协作需要运行在一台机器上，如果调度器先把 Nginx 调度到 Node1，因为资源不够，LogCollector 实际上是没办法调度到 Node1 上的，得重新再发起新的调度。

虽然能通过新一轮的调度，最终解决，但你思考：假如有几千个 Node 节点、数以万计的容器呢？解决这种协同调度：

- 要么等待所有设置了亲和性约束的任务全部就绪，才开始统一调度。这是典型的成组调度的解法，但也带来新的问题，调度效率会损失、资源无法充分利用、互相等待还有可能产生死锁。
- 要么就想办法提高单任务调度的效率，Google Omega 系统介绍过一种基于共享状态，通过乐观锁解决因并发导致资源冲突的方式，但方案无疑非常复杂。

将运行资源的需求声明定义在 Pod 上，直接以 Pod 为最小的原子单位来实现调度的话，Pod 与 Pod 之间不存在什么超亲密关系，如果非要有什么关系，就通过网络联系。

复杂的协同调度设计在 Kubernetes 中直接消失了。

## 7.2.8 容器的设计模式 Sidecar

通过组合两个不同角色的容器，共享资源，统一调度编排，在 Kubernetes 里面就是一个非常经典的容器设计模式 —— 即 Sidecar（边车）模式。

Sidecar 模式其实就是在 Pod 里面定义一些专门的容器，通过职责分离与容器的隔离特性，降低容器的复杂度。

通过图 7-6 所示的 Sidecar 容器（如日志记录、监控、安全性或数据同步 Sidecar 容器），能看到 Sidecar 模式通过增强或扩展主应用容器的功能，使开发一个高内聚、低耦合的软件变的更加容易。

:::center
  ![](../assets/sidecar.svg)<br/>
  图 7-6 容器中的边车设计模型
:::

本书第 8 章 8.2 节，笔者将通过代理型的 Sidecar 进一步阐述这种设计模式。

[^1]: 参见 https://static.googleusercontent.com/media/research.google.com/zh-CN//pubs/archive/41684.pdf