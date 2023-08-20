---
title: "Linux 内核追踪和 eBPF 介绍"
author: ["张俊(mingduo.zj@alibaba-inc.com)", "张俊(zj@opsnull.com)"]
date: 2023-08-12
lastmod: 2023-08-20T16:52:37+08:00
tags: ["ebpf"]
categories: ["ebpf"]
draft: false
series: ["ebpf"]
series_order: 1
---

eBPF 是当今热门的底层技术，在网络、安全、可观测性、云原生等场景得到广泛应用。

本文档以 Linux 追踪技术为基础，辅以 eBPF 的几个 Demo，介绍 eBPF 的原理、开发过程和应用场景。

<!--more-->


## <span class="section-num">1</span> 基础: Linux 内核追踪 {#基础-linux-内核追踪}

Linux 追踪是对运行中的内核和应用程序进行稳定性、性能分析和诊断的技术。

分为三个层次：

1.  Event Sources：各种探针 （probe）和追踪点（tracepoint），是数据源；
2.  Tracing Frameworks：从各种数据源收集数据的框架；
3.  Front-End Frameworks：用户端性能追踪工具，或编写自定义追踪工具的框架；

{{< figure src="images/内核追踪技术/2023-07-20_21-26-08_screenshot.png" width="400" >}}


### <span class="section-num">1.1</span> 发展历史 {#发展历史}

{{< figure src="images/内核追踪技术/2023-07-20_21-25-15_screenshot.png" width="100%" height="100%" >}}

-   不是按照一个整体统一设计出来的，是在 _演化过程中_ 慢慢出现的;
-   不同的框架都有自己一套使用和操作数据源的方法，他们各自有自身适合的使用场景。


### <span class="section-num">1.2</span> 事件源 {#事件源}

Event Sources：各种探针 （probe）和追踪点（tracepoint），是基础的数据源；

1.  kprobes:
    -   内核级动态追踪技术;
    -   可以在内核任意位置插桩的动态追踪技术;
    -   基于中断机制，通过在探测位置埋入中断指令来触发中断，在中断处理的过程中执行用户指定的钩子函数。

2.  uprobes:
    -   用户级动态追踪技术;
    -   在用户态和内核态的 kprobe 对应，能实现在函数入口、特定偏移处以及函数返回处进行插桩;

3.  tracepoints:
    -   内核级静态追踪技术；
    -   相对于 kprobe 接口更稳定，它被写好在指定的函数中，只要该函数在，内核的跟新不会导致tracepoints
        失效。
    -   tracepoints 能实现的功能，kprobe 也能实现。tracepoints 可以看作是定制化的内核探针，它不会像kprobe 一样随着函数名的改变或者函数地址的改变而失效，

4.  USDT:
    -   User-level Statically Defined Tracing, 用户态预定义静态跟踪，用户态下的 tracepoints。
    -   用户级静态追踪技术, 在用户程序的关键位置留下插桩点，用于对程序进行调试、性能观测等功能。

5.  PMU:
    -   PMU（Performance Monitoring Unit）即性能监控单元, 是一种硬件机制，并且它是为性能而生的。
    -   用于统计系统中发生的特定硬件事件，例如缓存未命中（Cache Miss）或者分支预测错误（Branch
        Misprediction）等。

tracepoint 定义示例：

```c
/*
 * Tracepoint for kernel_clone:
 */
TRACE_EVENT(sched_process_fork,

  TP_PROTO(struct task_struct *parent, struct task_struct *child),

  TP_ARGS(parent, child),

  TP_STRUCT__entry(
    __array(  char, parent_comm,  TASK_COMM_LEN )
    __field(  pid_t,  parent_pid      )
    __array(  char, child_comm, TASK_COMM_LEN )
    __field(  pid_t,  child_pid     )
  ),

  TP_fast_assign(
    memcpy(__entry->parent_comm, parent->comm, TASK_COMM_LEN);
    __entry->parent_pid = parent->pid;
    memcpy(__entry->child_comm, child->comm, TASK_COMM_LEN);
    __entry->child_pid  = child->pid;
  ),

  TP_printk("comm=%s pid=%d child_comm=%s child_pid=%d",
    __entry->parent_comm, __entry->parent_pid,
    __entry->child_comm, __entry->child_pid)
);
```


### <span class="section-num">1.3</span> 追踪框架 {#追踪框架}

-   In-Tree：ftrace，perf_event， eBPF;
-   Out-Tree: SystemTap，LTTng，DTrace，sysdig

    {{< figure src="images/基础:_Linux_内核追踪/2023-07-21_13-57-06_screenshot.png" width="100%" height="100%" >}}

    From：[RHEL 8 性能监控选项概述](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/8/html/monitoring_and_managing_system_status_and_performance/overview-of-performance-monitoring-options_monitoring-and-managing-system-status-and-performance)


#### ftrace {#ftrace}

{{< figure src="images/内核追踪技术/2023-07-20_21-28-25_screenshot.png" width="400" >}}

用户接口：tracefs 文件系统：

```shell
root@lima-ebpf-dev:/# ls /sys/kernel/debug/tracing/
README                      buffer_size_kb         enabled_functions
available_events            buffer_total_size_kb   error_log
available_filter_functions  current_tracer         events
available_tracers           dyn_ftrace_total_info  free_buffer
buffer_percent              dynamic_events         function_profile_enabled
```

可以追踪的事件：分门别类放在 `/sys/kerne/debug/tracing/events` 目录下：

```shell
root@lima-ebpf-dev:/# ls /sys/kernel/debug/tracing/events/syscalls/sys_enter_execve
enable  filter  format  hist  id  inject  trigger
```

-   开启： `"echo 1 > enable"` 即可开启该 tracepoint；
-   查看： `/sys/kernel/debug/tracing/trace_pipe` 中就可以看到输出结果了。

<!--listend-->

```shell
root@lima-ebpf-dev:/sys/kernel/debug/tracing/events/syscalls/sys_enter_execve# echo 1 > enable
root@lima-ebpf-dev:/sys/kernel/debug/tracing/events/syscalls/sys_enter_execve# cat /sys/kernel/debug/tracing/trace_pipe
           clear-285758  [000] ..... 687231.120698: sys_execve(filename: 559dae079f20, argv: 559dae079b20, envp: 559dae0550b0)
            sudo-285759  [003] ..... 687240.034196: sys_execve(filename: 55b93e3e3e00, argv: 55b93e389900, envp: 55b93e399cd0)
              ls-285760  [003] ..... 687243.634162: sys_execve(filename: 559dae098540, argv: 559dae098470, envp: 559dae0550b0)
            bash-285762  [001] ..... 687244.443490: sys_execve(filename: 5563d3de4dc8, argv: 5563d3dd69d0, envp: 5563d3de97e0)
          groups-285763  [002] ..... 687244.450417: sys_execve(filename: 55cc2c6a84e0, argv: 55cc2c6a9110, envp: 55cc2c6a5600)
        lesspipe-285764  [002] ..... 687244.456961: sys_execve(filename: 55cc2c6a7cd0, argv: 55cc2c6a6ff0, envp: 55cc2c6a5c60)
        basename-285765  [003] ..... 687244.458716: sys_execve(filename: 55ae91cc3a08, argv: 55ae907527c0, envp: 55ae91cc3948)
         dirname-285767  [001] ..... 687244.463996: sys_execve(filename: 55ae91ccf988, argv: 55ae91ccf8b0, envp: 55ae91ccf8c8)
       dircolors-285768  [002] ..... 687244.466723: sys_execve(filename: 55cc2c6a6270, argv: 55cc2c6ae0a0, envp: 55cc2c6a5c60)
           <...>-285769  [003] ..... 687247.664769: sys_execve(filename: 559dae079b00, argv: 559dae098560, envp: 559dae0550b0)
```

除了 tracepoint 外，kprobe 等也可以基于 tracefs 文件系统来实现，以 vfs_read 为例：

```text
ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos);
```

1.  先切换目录到 `/sys/kernel/debug/tracing` 下。
2.  向 kprobe_events 目录按照一定格式写入 kprobe 信息： `echo "p:test_probe vfs_read file=%di" >
       kprobe_events`
3.  内核会创建一个名为 `events/kprobes/test_probe/` 的目录(一个 tracepoint 实例，目录层级和内容和
    tracepoint 一致）
    ```shell
       # ls events/kprobes/test_probe/
       enable  filter  format  hist  id  trigger
    ```
4.  启用 kprobe： `echo 1 > events/kprobes/test_probe/enable`
5.  数据将输出到 tracefs 的 ring buffer，也就是 trace_pipe 文件中。

<!--listend-->

```shell
# cat trace_pipe
...
sshd-4854  [004] .... 5647382.743505: test_probe: (vfs_read+0x0/0x130) file=0xffffa0b0e87e7700
cat-15149 [009] .... 5647382.743515: test_probe: (vfs_read+0x0/0x130) file=0xffffa0b117fe2400
sshd-4854  [004] .... 5647382.743523: test_probe: (vfs_read+0x0/0x130) file=0xffffa0b0e87e7700
...
```


#### perf_event {#perf-event}

-   内核 2.6.31 版本开始支持（2009），用户空间使用 perf 命令。
-   唯一支持硬件性能计数（PMC）的技术。

{{< figure src="images/内核追踪技术/2023-07-20_21-28-53_screenshot.png" width="400" >}}

系统调用接口：perf_event_open

-   type 参数可以指定各种采集类型：PERF_TYPE_HARDWARE ，PERF_TYPE_SOFTWARE，PERF_TYPE_TRACEPOINT：

<!--listend-->

```c
int syscall(SYS_perf_event_open, struct perf_event_attr *attr,
           pid_t pid, int cpu, int group_fd, unsigned long flags);

 struct perf_event_attr {
        __u32 type;                 /* Type of event */
        __u32 size;                 /* Size of attribute structure */
        __u64 config;               /* Type-specific configuration */
。。。
```

perf list：查看支持的事件列表

```shell
[root@devops-110 ~]# perf list |& head
  alignment-faults                                   [Software event]
  bpf-output                                         [Software event]
  context-switches OR cs                             [Software event]
  cpu-clock                                          [Software event]
  cpu-migrations OR migrations                       [Software event]
  dummy                                              [Software event]
  emulation-faults                                   [Software event]
  major-faults                                       [Software event]
  minor-faults                                       [Software event]
  page-faults OR faults                              [Software event]
[root@devops-110 ~]#

[root@devops-110 ~]# perf list software

List of pre-defined events (to be used in -e):

  alignment-faults                                   [Software event]
  bpf-output                                         [Software event]
  context-switches OR cs                             [Software event]
  cpu-clock                                          [Software event]
  cpu-migrations OR migrations                       [Software event]
  dummy                                              [Software event]
  emulation-faults                                   [Software event]
  major-faults                                       [Software event]
  minor-faults                                       [Software event]
  page-faults OR faults                              [Software event]
  task-clock                                         [Software event]

[root@devops-110 ~]# perf list tracepoint |head
  block:block_bio_backmerge                          [Tracepoint event]
  block:block_bio_bounce                             [Tracepoint event]
  block:block_bio_complete                           [Tracepoint event]
  block:block_bio_frontmerge                         [Tracepoint event]
  block:block_bio_queue                              [Tracepoint event]
  block:block_bio_remap                              [Tracepoint event]
  block:block_dirty_buffer                           [Tracepoint event]
  block:block_getrq                                  [Tracepoint event]
  block:block_plug                                   [Tracepoint event]
  block:block_rq_abort                               [Tracepoint event]
[root@devops-110 ~]#
```

perf top: 查看当前 CPU 使用率

```c
$ perf top -e   sched:sched_wakeup --stdio

   PerfTop:      90 irqs/sec  kernel:100.0%  exact:  0.0% lost: 0/0 drop: 0/0 [1Hz sched:sched_wakeup],  (all, 4 CPUs)
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

    50.72%  comm=perf pid=285868 prio=120 target_cpu=003
     7.09%  comm=containerd pid=1454 prio=120 target_cpu=000
```

进程 CPU 统计：

```shell
[root@devops-110 ~]# perf stat ls

 Performance counter stats for 'ls':

              0.69 msec task-clock                #    0.734 CPUs utilized
                 0      context-switches          #    0.000 K/sec
                 0      cpu-migrations            #    0.000 K/sec
               265      page-faults               #    0.386 M/sec
   <not supported>      cycles
   <not supported>      instructions
   <not supported>      branches
   <not supported>      branch-misses

       0.000934209 seconds time elapsed

       0.000000000 seconds user
       0.000978000 seconds sys
```

perf record：统计 CPU Cycles（结果写入 `perf.data` 文件）：

```shell
[root@devops-110 ~]# perf record -e cycles  find /
[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 0.091 MB perf.data (2195 samples) ]
```

查看性能结果：

```shell
perf report
```

制作火焰图：

-   perf script | ./stackcollapse-perf.pl | ./flamegraph.pl &gt; perf-kernel.svg

{{< figure src="images/基础:_linux_追踪/2023-07-21_10-42-00_screenshot.png" width="400" >}}

更多参考：[perf Examples](https://www.brendangregg.com/perf.html)


#### eBPF {#ebpf}

{{< figure src="images/内核追踪技术/2023-07-20_21-30-35_screenshot.png" width="400" >}}

eBPF（extended Berkeley Packet Filter） 是事件驱动的，高性能，安全的内核级虚拟机。

-   4.x 开始支持；
-   最开始是 filtering packaget，当前应用场景扩展到：
    -   可观测性： 动态追踪（tracing），静态追踪（probe），性能分析（profiling）
    -   高性能网络；
    -   安全审计；

eBPF 工作流程：

1.  编写 eBPF 程序：内核态一般是 C + libpbf，用户态各种语言框架（C libbpf、Go cilium/ebpg、Rust Aya等）
2.  将 eBPF load 进内核，内核对 eBPF 程序进行各种 verify；
3.  将 eBPF attach 到各种数据源：kprobe/uprobe/tracepoint/USDT-probe；
4.  eBPF 程序将数据写入 eBPF map/ftrace/per buffer 等；
5.  用户态 pull 写入的数据；

eBPF 当前支持的各种数据源：

{{< figure src="images/内核追踪技术/2023-07-20_21-30-57_screenshot.png" width="400" >}}

对比传统的 perf_event profiling 和当前的 eBPF profiling 技术：

-   per_event: 需要将 event data dump 到 disk file 来离线分析；
-   eBPF: 直接基于虚拟机的 in-kernel programmable tracer；
-   性能，开销、可编程性革命！

{{< figure src="images/内核追踪技术/2023-07-20_21-31-13_screenshot.png" width="400" >}}

eBPF 为当前的各种 tracing framework， 如 ftrace、perf_vents 等，提供了 in-kernel 可编程能力；


### <span class="section-num">1.4</span> 用户前端 {#用户前端}

ftrace：

1.  内核虚拟文件系统，/sys/kernel/debug/tracing；
2.  trace-cmd: 命令行工具；

perf_event：

1.  perf：perf_event 专用的命令行工具；
2.  perf-tools: ftrace + perf_event 的前端封装工具集；

eBPF：

1.  bcc
2.  bpftrace

[BCC：](https://github.com/iovisor/bcc/blob/master/README.md)

-   eBPF 程序开发和运行框架 + 一些内核分析和可观测工具命令；
-   C + Python 混合开发；

{{< figure src="images/内核追踪技术/2023-07-20_21-33-30_screenshot.png" width="400" >}}

BCC 项目工具：

{{< figure src="images/内核追踪技术/2023-07-20_21-33-53_screenshot.png" width="400" >}}

[bpftrace：](https://github.com/iovisor/bpftrace)

-   eBPF 专用的高级追踪语言（high-level tracing language），类比：文本处理的 awk 工具；
-   底层基于 BCC，将命令行参数指定的追踪指令编译为 eBPF 字节码，注入内核执行。
-   内核追踪和性能分析的瑞士军刀：
    -   不用写 C 代码，不受限于别人已经提供的工具；
    -   高级 DSL 语言，非常适合简短的、一次性的内核性能分析场景；

{{< figure src="images/内核追踪技术/2023-07-20_21-34-34_screenshot.png" width="400" >}}

内部实现：

{{< figure src="images/内核追踪技术/2023-07-20_21-34-45_screenshot.png" width="400" >}}

示例：

```shell
# 列出 probe
root@lima-ebpf-dev:/Users/zhangjun/docs# bpftrace -l 'tracepoint:syscalls:sys_enter_*' |head
tracepoint:syscalls:sys_enter_accept
tracepoint:syscalls:sys_enter_accept4
tracepoint:syscalls:sys_enter_access
tracepoint:syscalls:sys_enter_acct
tracepoint:syscalls:sys_enter_add_key
```

bpftrace 脚本：

```shell
root@lima-ebpf-dev:/Users/zhangjun/docs# cat read.bt
 tracepoint:syscalls:sys_enter_read
 {
   @start[tid] = nsecs;
 }

 tracepoint:syscalls:sys_exit_read / @start[tid] /
 {
   @times = hist(nsecs - @start[tid]);
   delete(@start[tid]);
 }
root@lima-ebpf-dev:/Users/zhangjun/docs#

root@lima-ebpf-dev:/Users/zhangjun/docs# bpftrace ./read.bt
Attaching 2 probes...

^C

@start[286210]: 494881661648171
@start[1472]: 494893376210298
@start[285769]: 494898639861704
@times:
[4K, 8K)             224 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                 |
[8K, 16K)            331 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[16K, 32K)            70 |@@@@@@@@@@                                          |
[32K, 64K)            17 |@@                                                  |
[64K, 128K)            8 |@                                                   |
[128K, 256K)          18 |@@                                                  |
[256K, 512K)          36 |@@@@@                                               |
[512K, 1M)             4 |                                                    |
[1M, 2M)              13 |@@                                                  |
[2M, 4M)               5 |                                                    |
[4M, 8M)               1 |                                                    |
[8M, 16M)              0 |                                                    |
[16M, 32M)             1 |                                                    |
[32M, 64M)             0 |                                                    |
[64M, 128M)            0 |                                                    |
[128M, 256M)           0 |                                                    |
[256M, 512M)           0 |                                                    |
[512M, 1G)             0 |                                                    |
[1G, 2G)               1 |                                                    |
[2G, 4G)               2 |                                                    |

root@lima-ebpf-dev:/Users/zhangjun/docs#
```

更多参考：[The bpftrace One-Liner Tutorial](https://github.com/iovisor/bpftrace/blob/master/docs/tutorial_one_liners.md)

[BCC/bpftrace/libbpf 前景](https://www.brendangregg.com/blog/2020-11-04/bpf-co-re-btf-libbpf.html)：

The future of BPF performance tools, BCC Python, and bpftrace

For BPF performance tools, you should start with `running BCC and bpftrace tools`, and then `coding in bpftrace` .

The BCC tools should eventually be `switched from Python to libbpf C` under the hood, but will work the
same. Coding performance tools in `BCC Python is now considered deprecated as we =move to libbpf C with BTF and
CO-RE` (although we still have library work to do, such as for USDT support, so the Python versions will be
needed for a while). Note that there are other use cases of BCC that may continue to use the Python interface;
BPF co-maintainer Alexei Starovoitov and myself briefly discussed this on iovisor-dev.


### <span class="section-num">1.5</span> 总结 {#总结}

{{< figure src="images/内核追踪技术/2023-07-20_20-54-53_screenshot.png" width="400" >}}

FROM：<https://mmi.hatenablog.com/entry/2018/03/04/052249>


## <span class="section-num">2</span> eBPF 介绍 {#ebpf-介绍}


### <span class="section-num">2.1</span> Demo1: XDP {#demo1-xdp}

一个丢弃网络接口包的 Demo：

```c
#include "vmllinux.h"
#include <linux/bpf.h>

#define SEC(NAME) __attribute__((section(NAME), used))

SEC("xdp")
int xdp_drop_the_world(struct xdp_md *ctx) {
    return XDP_DROP;
}

char _license[] SEC("license") = "GPL";
```

​执行步骤：

```shell
# 编译为 eBPF 字节码
root@lima-ebpf-dev:~/demo# clang -O2 -target bpf -D __TARGET_ARCH_x86 -c xdp.c -o xdp.o

# 加载执行
root@lima-ebpf-dev:~/demo# ip link set dev lo xdp obj xdp.o sec xdp verbose
libbpf: loading xdp.o
libbpf: elf: section(3) xdp, size 16, link 0, flags 6, type=1
libbpf: sec 'xdp': found program 'xdp_drop_the_world' at insn offset 0 (0 bytes), code size 2 insns (16 bytes)
libbpf: elf: section(4) license, size 4, link 0, flags 3, type=1
libbpf: license of xdp.o is GPL
libbpf: elf: section(6) .symtab, size 96, link 1, flags 0, type=2
libbpf: looking for externs among 4 symbols...
libbpf: collected 0 externs total

# lo 不通
root@lima-ebpf-dev:~/demo# ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
^C
--- 127.0.0.1 ping statistics ---
6 packets transmitted, 0 received, 100% packet loss, time 5129ms

# 卸载
root@lima-ebpf-dev:~/demo# ip link set dev lo xdp off

# lo 通
root@lima-ebpf-dev:~/demo# ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=1.41 ms
^C
--- 127.0.0.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.413/1.413/1.413/0.000 ms
```


### <span class="section-num">2.2</span> Demo2: 内核静态追踪  traepoint 系统调用 write {#demo2-内核静态追踪-traepoint-系统调用-write}

```c
// minimal.bpf.c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

char LICENSE[] SEC("license") = "Dual BSD/GPL";

int my_pid = 0;

SEC("tracepoint/syscalls/sys_enter_write")
int handle_tp(void *ctx)
{
        int pid = bpf_get_current_pid_tgid() >> 32;

        if (pid != my_pid)
                return 0;

        bpf_printk("BPF triggered from PID %d.\n", pid);

        return 0;
}


//  minimal.c
#include <stdio.h>
#include <unistd.h>
#include <sys/resource.h>
#include <bpf/libbpf.h>
#include "minimal.skel.h"

static int libbpf_print_fn(enum libbpf_print_level level, const char *format, va_list args)
{
        return vfprintf(stderr, format, args);
}

int main(int argc, char **argv)
{
        struct minimal_bpf *skel;
        int err;

        /* Set up libbpf errors and debug info callback */
        libbpf_set_print(libbpf_print_fn);

        /* Open BPF application */
        skel = minimal_bpf__open();
        if (!skel) {
                fprintf(stderr, "Failed to open BPF skeleton\n");
                return 1;
        }

        /* ensure BPF program only handles write() syscalls from our process */
        skel->bss->my_pid = getpid();

        /* Load & verify BPF programs */
        err = minimal_bpf__load(skel);
        if (err) {
                fprintf(stderr, "Failed to load and verify BPF skeleton\n");
                goto cleanup;
        }

        /* Attach tracepoint handler */
        err = minimal_bpf__attach(skel);
        if (err) {
                fprintf(stderr, "Failed to attach BPF skeleton\n");
                goto cleanup;
        }

        printf("Successfully started! Please run `sudo cat /sys/kernel/debug/tracing/trace_pipe` "
               "to see output of the BPF programs.\n");

        for (;;) {
                /* trigger our BPF program */
                fprintf(stderr, ".");
                sleep(1);
        }

cleanup:
        minimal_bpf__destroy(skel);
        return -err;
}
```

执行：

```shell
root@lima-ebpf-dev:~/demo/libbpf-bootstrap/examples/c# ./minimal

root@lima-ebpf-dev:~/demo/libbpf-bootstrap/examples/c# cat /sys/kernel/debug/tracing/trace_pipe
           <...>-287833  [000] ..... 691417.240672: sys_execve(filename: 5591f151e430, argv: 5591f151e470, envp: 5591f14f80b0)

```


### <span class="section-num">2.3</span> Demo3: 内核动态追踪 kprobe 内核函数 do_unlinkat {#demo3-内核动态追踪-kprobe-内核函数-do-unlinkat}

```c
// kprobe.bpf.c
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>
#include <bpf/bpf_core_read.h>

char LICENSE[] SEC("license") = "Dual BSD/GPL";

SEC("kprobe/do_unlinkat")
int BPF_KPROBE(do_unlinkat, int dfd, struct filename *name)
{
        pid_t pid;
        const char *filename;

        pid = bpf_get_current_pid_tgid() >> 32;
        filename = BPF_CORE_READ(name, name);
        bpf_printk("KPROBE ENTRY pid = %d, filename = %s\n", pid, filename);
        return 0;
}

SEC("kretprobe/do_unlinkat")
int BPF_KRETPROBE(do_unlinkat_exit, long ret)
{
        pid_t pid;

        pid = bpf_get_current_pid_tgid() >> 32;
        bpf_printk("KPROBE EXIT: pid = %d, ret = %ld\n", pid, ret);
        return 0;
}


// kprobe.c
#include <stdio.h>
#include <unistd.h>
#include <signal.h>
#include <string.h>
#include <errno.h>
#include <sys/resource.h>
#include <bpf/libbpf.h>
#include "kprobe.skel.h"

static int libbpf_print_fn(enum libbpf_print_level level, const char *format, va_list args)
{
        return vfprintf(stderr, format, args);
}

static volatile sig_atomic_t stop;

static void sig_int(int signo)
{
        stop = 1;
}

int main(int argc, char **argv)
{
        struct kprobe_bpf *skel;
        int err;

        /* Set up libbpf errors and debug info callback */
        libbpf_set_print(libbpf_print_fn);

        /* Open load and verify BPF application */
        skel = kprobe_bpf__open_and_load();
        if (!skel) {
                fprintf(stderr, "Failed to open BPF skeleton\n");
                return 1;
        }

        /* Attach tracepoint handler */
        err = kprobe_bpf__attach(skel);
        if (err) {
                fprintf(stderr, "Failed to attach BPF skeleton\n");
                goto cleanup;
        }

        if (signal(SIGINT, sig_int) == SIG_ERR) {
                fprintf(stderr, "can't set signal handler: %s\n", strerror(errno));
                goto cleanup;
        }

        printf("Successfully started! Please run `sudo cat /sys/kernel/debug/tracing/trace_pipe` "
               "to see output of the BPF programs.\n");

        while (!stop) {
                fprintf(stderr, ".");
                sleep(1);
        }

cleanup:
        kprobe_bpf__destroy(skel);
        return -err;
}
```

编译执行：

```shell
root@lima-ebpf-dev:~/demo/libbpf-bootstrap/examples/c# make kprobe
  BPF      .output/kprobe.bpf.o
  GEN-SKEL .output/kprobe.skel.h
  CC       .output/kprobe.o
  BINARY   kprobe

root@lima-ebpf-dev:~/demo/libbpf-bootstrap/examples/c# ./kprobe


root@lima-ebpf-dev:~# touch test
root@lima-ebpf-dev:~# rm test

root@lima-ebpf-dev:~/demo/libbpf-bootstrap/examples/c# cat /sys/kernel/debug/tracing/trace_pipe
           <...>-287909  [002] d...1 691657.455453: bpf_trace_printk: KPROBE ENTRY pid = 287909, filename = test

           <...>-287909  [002] d...1 691657.455546: bpf_trace_printk: KPROBE EXIT: pid = 287909, ret = 0

```


### <span class="section-num">2.4</span> BPF 1992: Berkeley Packet Filter {#bpf-1992-berkeley-packet-filter}

-   BPF：Berkeley Packet Filter, 即伯克利报文过滤器。
-   源于 1992 的一篇论文：[《The BSD packet filter: A New architecture for user-level packet capture》](https://www.tcpdump.org/papers/bpf-usenix93.pdf)
-   最初在 BSD 系统实现，后来引入 Linux；

{{< figure src="images/eBPF_介绍/2023-07-21_11-48-39_screenshot.png" width="400" >}}

BPF 架构原理图（来自论文）：

1.  BPF是作为内核报文传输路径的一个旁路存在的，当报文到达内核驱动程序后，内核在将报文上送协议栈的同时，会额外将报文的一个副本交给 BPF。
2.  报文会经过 BPF 内部逻辑的过滤(这个逻辑可以自己设置 ) ， 然后最终送给用户程序 ( 比如 tcpdump)。

{{< figure src="images/eBPF_介绍/2023-07-21_11-49-20_screenshot.png" width="400" >}}

BPF 被广泛用于网络包过滤场景：

```shell
root@lima-ebpf-dev:~# tcpdump -d host 127.0.0.1 and port 80
Warning: assuming Ethernet
(000) ldh      [12]
(001) jeq      #0x800           jt 2    jf 18
(002) ld       [26]
(003) jeq      #0x7f000001      jt 6    jf 4
(004) ld       [30]
(005) jeq      #0x7f000001      jt 6    jf 18
(006) ldb      [23]
(007) jeq      #0x84            jt 10   jf 8
(008) jeq      #0x6             jt 10   jf 9
(009) jeq      #0x11            jt 10   jf 18
(010) ldh      [20]
(011) jset     #0x1fff          jt 18   jf 12
(012) ldxb     4*([14]&0xf)
(013) ldh      [x + 14]
(014) jeq      #0x50            jt 17   jf 15
(015) ldh      [x + 16]
(016) jeq      #0x50            jt 17   jf 18
(017) ret      #262144
(018) ret      #0
root@lima-ebpf-dev:~#
```

BPF 解决了什么问题 ？

-   安全、高效、灵活的执行包过滤：
    -   安全：内核虚拟机执行过滤指令，不会 crash 内核；
    -   高效：内核态直接操作，性能高（by user-kernel 数据拷贝、上下文切换）；
    -   灵活：用户可以自定义各种过滤规则；


### <span class="section-num">2.5</span> eBPF 2013+ {#ebpf-2013-plus}

Extended BPF（eBPF）：

1.  基础扩展：

{{< figure src="images/eBPF_介绍/2023-07-21_11-57-25_screenshot.png" width="400" >}}

1.  指令集和虚拟机扩展：

{{< figure src="images/eBPF_介绍/2023-07-21_11-57-46_screenshot.png" width="400" >}}

1.  基础设施扩展：
2.  BPF存储对象：BPF map
3.  BPF辅助函数（helper function）：可以更方便地利用内核功能或与内核交互
4.  尾调用（tail call）：高效地调用其他 BPF 程序
5.  安全加固原语（security hardening primitives）：bpf_spin_lock(lock)、bpf_spin_unlock(lock)
6.  用于钉住（pin）对象（例如 map、程序）的伪文件系统：BPF sysfs(sys/fs/bpf)
7.  LLVM 提供了一个 BPF 后端（back end）：因此使用 clang 这样的工具就可以将 C 代 码编译成BPF 对象文件（object file），最后生成BPF字节码


### <span class="section-num">2.6</span> eBPF 2023 {#ebpf-2023}

eBPF 成为通用内核执行环境：

-   热加载的方式动态的获取、修改内核中的关键数据和执行逻辑；
-   避免内核模块的方式可能会引入宕机风险；
-   具备堪比原生代码的执行效率。
-   新的软件平台；

用户态，内核态和 eBPF：

{{< figure src="images/eBPF_介绍/2023-07-21_12-15-02_screenshot.png" width="400" >}}

eBPF as a platform：全栈可编程平台：

-   可观测：kprobe, tracepoint
-   安全：secomp
-   高性能网络：sched, XDP

{{< figure src="images/eBPF_介绍/2023-07-20_19-56-53_screenshot.png" width="400" >}}


## <span class="section-num">3</span> eBPF 原理 {#ebpf-原理}


### <span class="section-num">3.1</span> eBPF 运行时架构 {#ebpf-运行时架构}

{{< figure src="images/eBPF_原理/2023-07-21_12-29-30_screenshot.png" width="400" >}}


### <span class="section-num">3.2</span> 开发和执行流程 {#开发和执行流程}

eBPF 程序的编译、加载和运行

{{< figure src="images/eBPF_原理/2023-07-21_12-36-24_screenshot.png" width="400" >}}


### <span class="section-num">3.3</span> bfp syscall {#bfp-syscall}

eBPF：在内核里运行自定义代码：

```shell
NAME
       bpf - perform a command on an extended BPF map or program

SYNOPSIS
       #include <linux/bpf.h>
       int bpf(int cmd, union bpf_attr *attr, unsigned int size);
```

man 2 bpf：

The bpf() system call performs a range of operations related to extended Berkeley Packet
Filters.  Extended BPF (or eBPF) is similar to the original ("classic") BPF (cBPF) used to
filter network packets.  For both cBPF and eBPF programs, the kernel <span class="underline">statically analyzes the
programs</span> before loading them, in order to ensure that they <span class="underline">cannot harm the running system</span>.

eBPF extends cBPF in multiple ways, including the ability to call a fixed set of <span class="underline">in-kernel
helper</span> functions (via the BPF_CALL opcode extension provided by eBPF) and access shared data
structures such as <span class="underline">eBPF maps</span>.

Generally, eBPF programs are <span class="underline">loaded by the user process</span> and automatically unloaded when the
process exits.  In some cases, for example, tc-bpf(8), the program will continue to stay
alive inside the kernel even after the process that loaded the program exits.  In that case,
the tc subsystem holds a reference to the eBPF program after the file descriptor has been
closed by the user-space program.  Thus, whether a specific program continues to live inside
the kernel depends on how it is further attached to a given kernel subsystem after it was
loaded via bpf().

Each eBPF program is <span class="underline">a set of instructions that is safe to run</span> until its completion.  An
in-kernel <span class="underline">verifier statically determines</span> that the eBPF program terminates and is safe to
execute.  During verification, the kernel in‐ crements reference counts for each of the maps
that the eBPF program uses, so that the attached maps can't be removed until the program is
unloaded.

eBPF programs can be <span class="underline">attached to different events</span>.  These events can be the arrival of
\_network packets, tracing events, classification events by network queueing disciplines (for
eBPF programs attached to a tc(8) classi‐ fier), and other types that may be added in the
future.

<span class="underline">A new event triggers execution of the eBPF program</span>, which may store information about the
event in eBPF maps.  Beyond storing data, eBPF programs may call a fixed set of in-kernel
helper functions.

通过 bpf syscall 来加载字节码程序和创建 Maps：

{{< figure src="images/eBPF_原理/2023-07-21_12-34-26_screenshot.png" width="400" >}}


### <span class="section-num">3.4</span> 编译 {#编译}

将 C 代码编译为 eBPF 字节码的过程：

1.  使用 LLVM 工具链；

{{< figure src="images/eBPF_原理/2023-07-20_20-01-05_screenshot.png" width="400" >}}

-   机器码(machine code)，学名机器语言指令，有时也被称为原生码（Native Code），是电脑的CPU可直接解读的数据(计算机只认识0和1)。
-   字节码（byte code）是一种包含执行程序、由一序列 OP代码(操作码)/数据对 组成的二进制文件。

字节码是一种中间码，它比机器码更抽象，需要直译器转译后才能成为机器码的中间代码。

字节码主要为了实现特定软件运行和软件环境、与硬件环境无关。字节码的实现方式是通过编译器和虚拟机器。编译器将源码编译成字节码，特定平台上的虚拟机器将字节码转译为可以直接执行的指令。

{{< figure src="images/eBPF_原理/2023-07-20_20-01-36_screenshot.png" width="400" >}}


### <span class="section-num">3.5</span> load &amp; verify {#load-and-verify}

verify：当 eBPF 程序被编译成字节码以后，然后使用加载程序 Loader 通过 bpf() 系统调用将字节码加载至内核。加载到 linux 内核后，会对程序进行验证。验证是非常重要的一步，验证时会对程序进行很多检查。如果没有这一步，攻击者就可以任意修改内核行为，导致系统被破坏。内核使用验证器(Verfier) 组件保证执行字节码的安全性，验证过程:

-   首先进行深度优先搜索，禁止循环；其他CFG验证。
-   以上一步生成的指令作为输入，遍历所有可能的执行路径。具体说就是模拟每条指令的执行，观察寄存器和栈的状态变化。

为了实现安全检查，对 eBPF 程序有如下限制：

-   eBPF程序不能调用任意的内核函数：
    -   只限于内核模块中列出的BPF辅助函数，函数支持列表也随着内核 的演进在不断增加;
    -   最新进展是支持了直接调用特定的内核函数调用
-   BPF程序不允许包含无法到达的指令，防止加载无效代码，延迟程序的终止
-   eBPF程序中循环次数限制且必须在有限时间内结束：
    -   Linux5.3在BPF中包含了对有界循环的支持，它有一个可验证的运行时间上限
-   eBPF堆栈大小被限制在MAX_BPF_STACK：
    -   截止到内核Linux5.8版本，被设置为512eBPF字节码大小最 初被限制为 4096 条指令，
    -   截止到内核 Linux 5.8 版本， 当前已将放宽至 100 万指令 ，对于无特权的 BPF 程序，仍然保留 4096 条限制 (BPF_MAXINSNS);
-   新版本的 eBPF 也支持了多个 eBPF 程序级联调用， 可以通过组合实现更加强大的功能

例子：程序代码第 14 行将指针改为空：

```c
SEC("xdp")

int xdp_drop_the_world(struct xdp_md *ctx) {

    // drop everything

    // 意思是无论什么网络数据包，都drop丢弃掉

    void *data = (void*)(long)ctx->data;

    void *data_end = (void *)(long)ctx->data_end;

    struct ethhdr *eth = data;

    if ((void*)eth + sizeof(*eth) <= data_end) {

        struct iphdr *ip = data + sizeof(*eth);

        if ((void*)ip + sizeof(*ip) <= data_end) {

            if (ip->protocol == IPPROTO_UDP) {

                struct udphdr *udp = (void*)ip + sizeof(*ip);

                if ((void*)udp + sizeof(*udp) <= data_end) {

                    udp = 0;

                    if (udp->dest == __builtin_bswap16(7999)) {

                        udp->dest = __builtin_bswap16(7998);

                    }

                }

            }

        }



    }

    return XDP_PASS;

}
```

加载报错信息：

```shell
lemon@lemon-server:~$ sudo ip link set dev lo xdp obj xdp-example.o sec xdp verbose~

libbpf: load bpf program failed: Permission denied

libbpf: -- BEGIN DUMP LOG ---

libbpf:

0: (61) r2 = *(u32 *)(r1 +0)

1: (61) r1 = *(u32 *)(r1 +4)

2: (bf) r3 = r2

3: (07) r3 += 14

4: (2d) if r3 > r1 goto pc+12

 R1_w=pkt_end(id=0,off=0,imm=0) R2_w=pkt(id=0,off=0,r=14,imm=0) R3_w=pkt(id=0,off=14,r=14,imm=0) R10=fp0

5: (bf) r3 = r2

6: (07) r3 += 34

7: (2d) if r3 > r1 goto pc+9

 R1_w=pkt_end(id=0,off=0,imm=0) R2_w=pkt(id=0,off=0,r=34,imm=0) R3_w=pkt(id=0,off=34,r=34,imm=0) R10=fp0

8: (71) r3 = *(u8 *)(r2 +23)

9: (55) if r3 != 0x11 goto pc+7

 R1=pkt_end(id=0,off=0,imm=0) R2=pkt(id=0,off=0,r=34,imm=0) R3=inv17 R10=fp0

10: (07) r2 += 42

11: (2d) if r2 > r1 goto pc+5

 R1=pkt_end(id=0,off=0,imm=0) R2_w=pkt(id=0,off=42,r=42,imm=0) R3=inv17 R10=fp0

12: (b7) r1 = 2

13: (69) r2 = *(u16 *)(r1 +0)

R1 invalid mem access 'inv'

processed 14 insns (limit 1000000) max_states_per_insn 0 total_states 1 peak_states 1 mark_read 1

libbpf: -- END LOG --

libbpf: failed to load program 'xdp_drop_the_world'

libbpf: failed to load object 'xdp-example.o'
```


### <span class="section-num">3.6</span> JIT {#jit}

验证完成后，进行 JIT 编译，将字节码转换成 CPU 可运行的机器码。

-   JIT 编译器可以极大加速 BPF 程序的执行，因为与解释器相比它们可以降低每个指令的开销
-   减少生成的可执行镜像的大小，因此对 CPU 的指令缓存更友好。

{{< figure src="images/eBPF_原理/2023-07-20_20-03-14_screenshot.png" width="400" >}}


### <span class="section-num">3.7</span> eBPF 程序的执行 {#ebpf-程序的执行}

{{< figure src="images/eBPF_原理/2023-07-20_20-03-38_screenshot.png" width="400" >}}

BPF程序指令何时执行?

-   eBPF 程序指令都是在内核的特定 Hook 点执行，不同类型的程序有不同的钩子，有不同的上下文(ctx)

将指令 load 到内核时，内核会创建 bpf_prog 存储指令，但只是第一步，成功运行这些指令还需要完成以下两个步骤：

-   将 bpf_prog 与内核中的特定 Hook 点关联起来，也就是将BPF程序挂到钩子上。
-   在 Hook 点被访问到时，取出 bpf_prog，执行这些指令。

举例：若某个kprobe探测点的内核地址attach了一段BPF程序后，当内核执行到这个地址时发生陷入(trap)，唤醒
kprobe 的回调函数，后者又会触发attach的BPF程序执行。


## <span class="section-num">4</span> eBPF 程序开发 {#ebpf-程序开发}


### <span class="section-num">4.1</span> 开发框架 {#开发框架}

-   eBPF Kernel 程序：C 语言开发，libbpfc 库；
-   eBPF User 程序：C/Go/Rust/Python 等；
    -   C： libbpf
    -   Go：cilium/ebpf， libbpfgo
    -   Python：bcc
    -   DSL： bpftrace
    -   Rust： ady

内核代码 samples/bpf 中也有很多示例程序可以学习参考。

```shell
$ ls ~/codes/kernel/linux-v4.19.91/samples/bpf/
Makefile                     map_perf_test_kern.c               sockex2_user.c         tcp_bufs_kern.c         test_current_task_under_cgroup_kern.c  trace_event_user.c   tracex7_user.c           xdp_redirect_map_user.c
README.rst                   map_perf_test_user.c               sockex3_kern.c         tcp_clamp_kern.c        test_current_task_under_cgroup_user.c  trace_output_kern.c  xdp1_kern.c              xdp_redirect_user.c
bpf_insn.h                   offwaketime_kern.c                 sockex3_user.c         tcp_cong_kern.c         test_ipip.sh*                          trace_output_user.c  xdp1_user.c              xdp_router_ipv4_kern.c
bpf_load.c                   offwaketime_user.c                 spintest_kern.c        tcp_iw_kern.c           test_lru_dist.c                        tracex1_kern.c       xdp2_kern.c              xdp_router_ipv4_user.c
bpf_load.h                   parse_ldabs.c                      spintest_user.c        tcp_rwnd_kern.c         test_lwt_bpf.c                         tracex1_user.c       xdp2skb_meta.sh*         xdp_rxq_info_kern.c
cookie_uid_helper_example.c  parse_simple.c                     syscall_nrs.c          tcp_synrto_kern.c       test_lwt_bpf.sh                        tracex2_kern.c       xdp2skb_meta_kern.c      xdp_rxq_info_user.c
cpustat_kern.c               parse_varlen.c                     syscall_tp_kern.c      test_cgrp2_array_pin.c  test_map_in_map_kern.c                 tracex2_user.c       xdp_adjust_tail_kern.c   xdp_sample_pkts_kern.c
cpustat_user.c               run_cookie_uid_helper_example.sh*  syscall_tp_user.c      test_cgrp2_attach.c     test_map_in_map_user.c                 tracex3_kern.c       xdp_adjust_tail_user.c   xdp_sample_pkts_user.c
fds_example.c                sampleip_kern.c                    task_fd_query_kern.c   test_cgrp2_attach2.c    test_overhead_kprobe_kern.c            tracex3_user.c       xdp_fwd_kern.c           xdp_tx_iptunnel_common.h
hash_func01.h                sampleip_user.c                    task_fd_query_user.c   test_cgrp2_sock.c       test_overhead_raw_tp_kern.c            tracex4_kern.c       xdp_fwd_user.c           xdp_tx_iptunnel_kern.c
lathist_kern.c               sock_example.c                     tc_l2_redirect.sh*     test_cgrp2_sock.sh*     test_overhead_tp_kern.c                tracex4_user.c       xdp_monitor_kern.c       xdp_tx_iptunnel_user.c
lathist_user.c               sock_example.h                     tc_l2_redirect_kern.c  test_cgrp2_sock2.c      test_overhead_user.c                   tracex5_kern.c       xdp_monitor_user.c       xdpsock.h
load_sock_ops.c              sock_flags_kern.c                  tc_l2_redirect_user.c  test_cgrp2_sock2.sh*    test_override_return.sh*               tracex5_user.c       xdp_redirect_cpu_kern.c  xdpsock_kern.c
lwt_len_hist.sh              sockex1_kern.c                     tcbpf1_kern.c          test_cgrp2_tc.sh*       test_probe_write_user_kern.c           tracex6_kern.c       xdp_redirect_cpu_user.c  xdpsock_user.c
lwt_len_hist_kern.c          sockex1_user.c                     tcp_basertt_kern.c     test_cgrp2_tc_kern.c    test_probe_write_user_user.c           tracex6_user.c       xdp_redirect_kern.c
lwt_len_hist_user.c          sockex2_kern.c                     tcp_bpf.readme         test_cls_bpf.sh*        trace_event_kern.c                     tracex7_kern.c       xdp_redirect_map_kern.c
$
```


### <span class="section-num">4.2</span> Event {#event}

﻿
eBPF 程序是事件驱动的，当事件发生后执行 eBPF 内核程序，常见事件类型：

-   system calls
-   tracepoint（静态）
-   kprobes（动态）
-   network package

事件体系本身也比较复杂，需要对内核知识有一定的了解。以 tracepoint 事件为例，通过 perf list
tracepoint可以看到内核 tracepoint：

```shell
root@lima-ebpf-dev:/Users/zhangjun/go/src/gitlab.alibaba-inc.com/apsara_paas/hello-ebpf/process# perf list tracepoint |& head
  alarmtimer:alarmtimer_cancel                       [Tracepoint event]
  alarmtimer:alarmtimer_fired                        [Tracepoint event]
  alarmtimer:alarmtimer_start                        [Tracepoint event]
  alarmtimer:alarmtimer_suspend                      [Tracepoint event]
  amd_cpu:amd_pstate_perf                            [Tracepoint event]
  avc:selinux_audited                                [Tracepoint event]
  block:block_bio_backmerge                          [Tracepoint event]
  block:block_bio_bounce                             [Tracepoint event]
  block:block_bio_complete                           [Tracepoint event]
  block:block_bio_frontmerge                         [Tracepoint event]
```


### <span class="section-num">4.3</span> Map {#map}

eBPF 的存储模块由 11 个 64 位寄存器、一个程序计数器和一个 512 字节的栈组成。这个模块用于控制 eBPF 程序的执行。其中，R0 寄存器用于存储函数调用和 eBPF 程序的返回值，这意味着函数调用最多只能有一个返回值；
R1-R5 寄存器用于函数调用的参数，因此函数调用的参数最多不能超过 5 个；而 R10 则是一个只读寄存器，用于从栈中读取数据。当使用较大的内存，需要用map。

Map 是 eBPF 的核心功能。内核运行的代码与加载其的用户空间程序可以通过 map 机制实现双向实时通信，这个特性非常有用，让内核态程序和用户态程序相互影响，有点类似 go 语言中的 channel。

BPF Map是驻留在内核中的以键/值方式存储的数据结构，可以被任何知道它们的 eBPF 程序访问。在用户空间运行的程序也可以通过使用文件描述符来访问eBPF Map。可以在eBPF Map中存储任何类型的数据，但需要指定个数和大小。在内核中，键和值都被视为二进制的方式来存储。

eBPF Map用于用户空间和内核空间之间进行双向的数据交换、信息传递。彼此共享MAP的BPF程序不需要具有相同的程序类型。

{{< figure src="images/eBPF_原理/2023-07-20_20-05-16_screenshot.png" width="400" >}}

Map 类型有：

-   Hash tables, Arrays
-   LRU (Least Recently Used)
-   Ring Buffer
-   Stack Trace
-   LPM (Longest Prefix match)
-   ...

更多详细内容参考 <https://yuque.antfin.com/qe7tfv/gf2vxx/xnkqatrg545utnmg%EF%BB%BF>

eBPF Map也有自己的 CRUD，主要操作如下：

-   bpf_map_create 函数，创建 Map
-   bpf_map_lookup_elem(map, key)函数，通过key查询BPF Map，得到对应value
-   bpf_map_update_elem(map, key, value, options)函数，通过key-value更新BPF Map，如果这个key不存在，也可以作为新的元素插入到BPF Map中去
-   bpf_map_get_next_key(map, lookup_key, next_key)函数，这个函数可以用来遍历BPF Map，下文有具体的介绍。


### <span class="section-num">4.4</span> help funcs {#help-funcs}

eBPF 程序并不能随意调用内核函数，因此，内核定义了一系列的辅助函数，用于 eBPF 程序与内核其他模块进行交互。限制对内核函数调用也是保证 eBPF 程序安全性的重要部分。

从内核 5.13 版本开始，部分内核函数（如 tcp_slow_start()、tcp_reno_ssthresh() 等）也可以被 BPF 程序直接调用了。不过，这些函数只能在 TCP 拥塞控制算法的 BPF 程序中调用。

查看所有helper函数：<https://man7.org/linux/man-pages/man2/bpf.2.html%EF%BB%BF>

{{< figure src="images/eBPF_原理/2023-07-20_20-06-53_screenshot.png" width="400" >}}

不同类型的 eBPF 程序所支持的辅助函数是不同的。比如，对于kprobe类型的 eBPF 程序，可以在命令行中执行
bpftool feature probe ，来查询当前系统支持的辅助函数列表：

﻿辅助函数类型包括：

-   生成随机数
-   获取时间
-   操作 eBPF Maps
-   获取进程上下文
-   操作网络报文
-   ....

关于 eBPF 的程序类型，我们后面做介绍。


### <span class="section-num">4.5</span> Program Type {#program-type}

eBPF 程序类型决定了一个 eBPF 程序可以挂载的事件类型和事件参数，这也就意味着，内核中不同事件会触发不同类型的eBPF 程序。程序类型根据挂载的事件确定。

根据内核头文件 bpf.h 中 bpf_prog_type 的定义，Linux 内核 v5.13 已经支持 30 种不同类型的 eBPF 程序（注意，BPF_PROG_TYPE_UNSPEC表示未定义）：

对于具体的内核来说，因为不同内核的版本和编译配置选项不同，一个内核并不会支持所有的程序类型。你可以在命令行中执行下面的命令，来查询当前系统支持的程序类型：

```text
bpftool feature probe | grep program_type
```

根据具体功能和应用场景的不同，这些程序类型大致可以划分为三类

1.  第一类是跟踪，即从内核和程序的运行状态中提取跟踪信息，来了解当前系统正在发生什么。
2.  第二类是网络，即对网络数据包进行过滤和处理，以便了解和控制网络数据包的收发过程。
3.  第三类是除跟踪和网络之外的其他类型，包括安全控制、BPF 扩展等等。
