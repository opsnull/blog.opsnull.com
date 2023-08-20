---
title: "Function Stack Unwinding"
author: ["opsnull"]
date: 2023-08-07T00:00:00+08:00
lastmod: 2023-08-20T16:51:08+08:00
tags: ["linux", "dwarf", "debug"]
categories: ["debug"]
draft: false
series: ["elf-debug"]
series_order: 6
---

介绍 Linux 函数调用栈生成和管理机制。

<!--more-->

stack unwinding 指的是获得当前函数调用栈的过程，当前有几种实现方式：

1.  FP：frame pointer；
2.  DWARF CFI（Call Frame Information），如 .eh_frame Section 信息；
3.  ORC： 4.14 及以后版本内核专用，简化版的 DWARF CFI；
4.  LBR： 新的 Intel CPU 支持；


## <span class="section-num">1</span> FP {#fp}

frame pointer 是通过一个特定的 CPU 寄存器(rbp)来保存栈指针，由编译器在函数调用和退出时添加额外的指令来 save、
setup 和 restore 该 CPU 寄存器中保存的 frame pointer。

{{< figure src="/images/FP/2023-08-09_10-47-53_screenshot.png" width="400" >}}

{{< figure src="/images/FP/2023-08-09_10-49-28_screenshot.png" width="400" >}}

From：

1.  <https://cs.wellesley.edu/~cs240/s16/slides/x86-procedures.pdf>
2.  <https://inst.eecs.berkeley.edu/~cs161/sp15/discussions/dis06-assembly.pdf>

基本原理：函数调用时，gcc 填充的指令会将函数参数 args、函数自动变量、函数返回地址、当前栈指针（rsp 寄存器中 保存）push 当前 frame stack，并将 frame stack 地址存入 rpb 寄存器。当前 frame 的函数返回地址位于 rbp+8 内存 中，通过查找二进制符号表（.symtab) 即可获得该地址的函数名称。这些细节是体系结构相关的 ABI 如 X86_64 ABI，来 标准化定义的。

-   参考：[Writing a Linux Debugger Part 8: Stack unwinding](https://blog.tartanllama.xyz/writing-a-linux-debugger-unwinding/)

frame pointer 是用户空间程序通用的 stack unwinding 机制，但是由于性能和开销问题，从 GCC 4.6 开始，各大发行版
(Centos7/Ubuntu 20.04)发布的软件包 `都默认关闭了 frame pointer` ，转而使用 DWARF 信息来做 stack unwinding。这会导致 bcc/bpftrace 等依赖 FP 来做 uprobe/usdt 的 stack unwinding 功能不可用，所以关闭了 FP 后，它们不能正常打印
ustack；

Fedora 38 开始[默认在编译软件包时开启 -fno-omit-frame-pointer](https://pagure.io/fesco/issue/2923) 编译器参数：

Golang [从 1.7 开始](https://github.com/golang/go/issues/15840)对 amd64 提供 frame pointer 的支持。（[internal-abi.md](https://go.googlesource.com/go/+/refs/heads/dev.regabi/src/cmd/compile/internal-abi.md))

测试程序：

```c
// https://medium.com/coccoc-engineering-blog/things-you-should-know-to-begin-playing-with-linux-tracing-tools-part-i-x-225aae1aaf13
#include <stdio.h>
#include <unistd.h>void func_d() {
                int msec=1;
                printf("%s","Hello world from D\n");
                usleep(10000*msec);
}
void func_c() {
                printf("%s","Hello from C\n");
                func_d();
}
void func_b() {
                printf("%s","Hello from B\n");
        func_c();
}
void func_a() {
                printf("%s","Hello from A\n");
                func_b();
}
int main() {
        func_a();
}
```

编译，确认 gcc 在函数开始插入了保存 fp 的指令：

```shell
# 编译（不开启优化，默认添加 FP）
root@lima-ebpf-dev:~# gcc test.c -o test
root@lima-ebpf-dev:~# objdump -S  test |grep func_c
000000000000119e <func_c>:
    11de:       e8 bb ff ff ff          call   119e <func_c>
root@lima-ebpf-dev:~# objdump -S  test |grep -A5 func_c
000000000000119e <func_c>:
    119e:       f3 0f 1e fa             endbr64
    11a2:       55                      push   %rbp
    11a3:       48 89 e5                mov    %rsp,%rbp # 将函数返回地址 push 到当前栈

    11a6:       48 8d 05 6a 0e 00 00    lea    0xe6a(%rip),%rax        # 2017 <_IO_stdin_used+0x17>
    11ad:       48 89 c7                mov    %rax,%rdi
--
    11de:       e8 bb ff ff ff          call   119e <func_c>
    11e3:       90                      nop
    11e4:       5d                      pop    %rbp
    11e5:       c3                      ret

00000000000011e6 <func_a>:
root@lima-ebpf-dev:~#
```

打印函数栈：

1.  使用 bpftrace 的 uprobe + ustack

<!--listend-->

```shell
root@lima-ebpf-dev:~# bpftrace -e 'uprobe:./hello:func_d {printf("%s",ustack)}' -c ./hello
Attaching 1 probe...
Hello from A
Hello from B
Hello from C
Hello world from D

        func_d+0
        func_b+33
        func_a+33
        main+18
        __libc_start_call_main+128


```

1.  使用 perf probe + record + script

<!--listend-->

```shell
root@lima-ebpf-dev:~# gcc test.c -g -o test # 保存 FP

root@lima-ebpf-dev:~# perf probe -x ./hello func_d  # uprobe
Added new event:
  probe_hello:func_d   (on func_d in /root/hello)

You can now use it in all perf tools, such as:

        perf record -e probe_hello:func_d -aR sleep 1

root@lima-ebpf-dev:~# perf record -e probe_hello:func_d -aR -g ./hello  # -g 默认使用 FP，也可以指定 -g --call-graph dwarf
Hello from A
Hello from B
Hello from C
Hello world from D
[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 0.162 MB perf.data (1 samples) ]
root@lima-ebpf-dev:~#

root@lima-ebpf-dev:~# perf script
hello 14239 [002]  5516.587824: probe_hello:func_d: (561112f12169)
            561112f12169 func_d+0x0 (/root/hello)
            561112f121e3 func_b+0x21 (/root/hello)
            561112f12207 func_a+0x21 (/root/hello)
            561112f1221c main+0x12 (/root/hello)
            7fe3e67e1d90 __libc_start_call_main+0x80 (/usr/lib/x86_64-linux-gnu/libc.so.6)
```


## <span class="section-num">2</span> GCC 优化和 FP {#gcc-优化和-fp}

从 GCC 4.6 开始，[只要开启了优化(从 -O/-O1 开始), 就会关闭 frame pointer](https://gcc.gnu.org/bugzilla//show_bug.cgi?id=100811)：

-O/-O1
: Optimize. Optimizing compilation takes somewhat more time, and a lot more memory for a large
    function. With ‘-O’, the compiler `tries to reduce code size and execution time`, without performing any
    optimizations that take a great deal of compilation time.

-O2
: Optimize even more. GCC performs nearly `all supported optimizations` that do not involve a space-speed
    tradeoff. As compared to ‘-O’, this option increases both compilation time and the performance of the
    generated code.

-O3
: Optimize yet more. ‘-O3’ turns on all optimizations specified by ‘-O2’ and also turns on the
    following optimization flags:

-Os
: Optimize for size. ‘-Os’ enables all ‘-O2’ optimizations except those that often increase code size:

-O0
: Reduce compilation time and make debugging produce the expected results. `This is the default`.

如果要开启 FP，则在使用 gcc 时：

1.  不开启优化，如不指定任何 -O 选项或指定 -O0；
2.  或者明确指定参数：-fno-omit-frame-pointer 或 --enable-frame-pointer;

-fomit-frame-pointer 解释：Omit the frame pointer in functions that don’t need one. This avoids the
instructions to save, set up and restore the frame pointer; on many targets it also makes an extra register
available.  On some targets this flag has no effect because the standard calling sequence always uses a frame
pointer, so it cannot be omitted.  Note that `‘-fno-omit-frame-pointer’` doesn’t guarantee the frame pointer is
used in all functions. Several targets always omit the frame pointer in leaf functions. `Enabled by default at
‘-O’ and higher`.

----

For x86-64, the ABI (PDF) `encourages the absence of a frame pointer`. The rationale is more or less "we have
DWARF now, so it's not necessary for debugging or exception unwinding; if we make it optional from day one,
then no software will come to depend on its existence."

x86-64 does have more registers than x86-32, but it `still doesn't have enough`. Freeing up more general-purpose
registers is always a Good Thing from a compiler's point of view. The operations that require a stack crawl
are slower, yes, but they are rare events, so it's a good tradeoff for shaving a few cycles off every
subroutine call plus fewer stack spills.


## <span class="section-num">3</span> DWARF CFI {#dwarf-cfi}

frame pinter 并不是运行程序所必须的, 而只在 debugger unwinding frame 时才需要。而编译器了解所有 func stack
frame 的大小和分配情况, 所以在编译时可以将 func stack frame 信息写入 elf 的 .debug_XX Sections 或 debuginfo 文件, 后续 debugger 通过读取这些内容来判断函数调用关系，这就出现了 DWARF CFI 规范.

[DWARF CFI（DWARF 3 Spec）](https://refspecs.linuxfoundation.org/LSB_3.0.0/LSB-PDA/LSB-PDA.junk/dwarfext.html) 是一种在 EFL 文件的 `.debug_frame` Section 中保存 unwind 信息的规范，通过解析这些信息，可以 unwind stack：

-   通过编译时添加 -g 选项来生成；
-   .debug_frame 不需要加载到内存，可以位于二进制外的单独 debuginfo 文件中；
-   Linus 反对内核自身使用 DWARF 来做 backtrace unwind 的邮件：<https://lkml.org/lkml/2012/2/10/356%EF%BC%8C%E6%89%80%E4%BB%A5> DWARF
    unwinder 只在用户空间程序使用；
-   使用 readelf -w XX 来查看 .debug_frame 的内容；

在编译的二进制不支持 stack frame pointer（gcc 开启优化有就关闭 FP，故各大发行版提供的软件包一般都不支持 FP）时，
gdb/perf 使用 debuginfo（DWARF 格式，一般使用 GNU libunwind 包，或 elfutils 提供的 libdw ）来进行 stack
unwinding/单步调试/内存地址和源文件对应关系。

在 CFI 的基础上，又提出了 [exception handler framework(.eh_frame)](https://refspecs.linuxfoundation.org/LSB_3.0.0/LSB-Core-generic/LSB-Core-generic/ehframechpt.html), 用来解决各种语言的结构对 reliable unwinding
的需求。 `.eh_frame` 是 .debug_frame 的子集，遵从 DWARF CFI Extensions 规范， `是 unwinding stack 专用的特性` （而
.debug_frame 功能更通用)：

-   `.eh_frame 会被 gcc 默认生成` ，不会被 strip，会被加载到内存（而 .debug_frame 等是会被 strip，默认不会被加载到内存）。
-   使用 readelf -Wwf XX 来查看 .eh_frame section 的内容；

C++ 在任意位置可能触发的异常也是通过 DWARF .eh_frame table（也称 unwind table） 来实现 Exception Stack 展示的。由于异常处理需要编译器插入额外的异常处理代码和调试信息，会导致结果二进制文件变大，所以 gcc 支持使用-fno-exceptions 和 -fno-unwind-tables 来关闭 C++ 的 Exception 功能。

对于 DWARF 的解析，主要有两个 library：

1.  `BFD (libbfd)` is used by the `GNU binutils`, including objdump which played a star role in this article, ld
    (the GNU linker) and as (the GNU assembler).
2.  `libdwarf` - which together with its big brother libelf are used for the tools on Solaris and FreeBSD
    operating systems.

对于 stack unwinding 支持，主要有两个库： `GNU libunwind` 和 `elfutils 的 libdw.so` ，它们都支持 FP 和 DWARF
unwinding：

1.  libundwind: 使用 debuginfo（DWARF 格式）需要安装 libunwind 库，这样在进程二进制没有编译支持 stack frame
    pointer 时，基于 DWARF 来进行 stack unwinding：
    -   需要 libunwind 1.1 及以上版本，且编译时配置了 --enable-debug-frame， CentOS 7 均支持；
        ```shell
             yum install libunwind libunwind-debuginfo -y
        ```
2.  libdw.so: elfutils 项目提供的 libdw.so 库也支持基于 DWARF 的 stack unwind 且性能比 libunwind 好一些。当前
    `perf/bpftrace` 等工具使用 libdw，[perf tools: Add libdw DWARF unwind support](https://lwn.net/Articles/579508/)
    -   Ubuntu/Debian 系统对应的是 libdw1 包。

<!--listend-->

```shell
# 17E：

# perf 依赖 libdw，由 elfutils-libs 包提供
# ldd /usr/bin/perf  |grep -E 'dw|unw'
        libdw.so.1 => /lib64/libdw.so.1 (0x00007f5a5cf5f000)
# rpm -qf /lib64/libdw.so.1
elfutils-libs-0.176-4.1.alios7.x86_64
```

参考：

1.  .eh_frame: [Reliable and Fast DWARF-Based Stack Unwinding](https://inria.hal.science/hal-02297690/file/main.pdf)


## <span class="section-num">4</span> Kernel ORC {#kernel-orc}

linus 没有接收使用 DWARF 作为 kernel 的 frame unwinder 机制，还是坚守 frame pointer 机制。但是从 Kernel v4.14
开始（CentOS 8 开始），内核开始使用 ORC unwinder 机制来作为 DWARF 的简化实现。

-   [x86: ORC unwinder (previously undwarf)](https://lwn.net/Articles/727553/)
-   [Unwinding a Stack by Hand with Frame Pointers and ORC](https://blogs.oracle.com/linux/post/unwinding-stack-frame-pointers-and-orc)

内核编译选项：

1.  CONFIG_FRAME_POINTER：编译开启了 frame pointers 的内核
    -   支持的版本： 2.6.9–2.6.39, 3.0–3.19, 4.0–4.20, 5.0–5.19, 6.0–6.4, 6.5-rc+HEAD
2.  CONFIG_UNWINDER_ORC: 编译开启了 ORC unwiner 的内核，用来 unwinding kernel stack traces。
    -   支持的版本：X86_64, 4.15–4.20, 5.0–5.19, 6.0–6.4, 6.5-rc+HEAD

<!--listend-->

```shell
# 17E 的情况：关闭了 FRAME_POINTER, 而是使用 ORC unwinder
# grep -E  'UNWINDER_ORC|FRAME_POINTER' /boot/config-4.19.91-007.ali4000.alios7.x86_64
CONFIG_SCHED_OMIT_FRAME_POINTER=y
CONFIG_UNWINDER_ORC=y
# CONFIG_UNWINDER_FRAME_POINTER is not set
```


## <span class="section-num">5</span> perf &amp; bfptrace {#perf-and-bfptrace}

perf record 支持捕获函数调用栈, 对于 `用户空间程序` ，可以使用 --call-grah 选项来指定 unwinding stack 的方法：

```text
perf record --call-graph method command
```

fp
: Uses the `frame pointer method`. Depending on compiler optimization, such as with binaries built with the
    GCC option --fomit-frame-pointer, this may not be able to unwind the stack.
    -   对于当前发行版自带的用户空间程序，由于当前发行版默认都开了优化，也就是默认开启了 --fomit-frame-pointer，所以该方法可能不适用；（但是如果是用户自己编译的程序，则可以设置 -fno-omit-frame-pointer 或
        --enable-frame-pointer 来开启 frame pointer）；

dwarf
: Uses `DWARF Call Frame Information` to unwind the stack.
    -   需要安装了可执行程序的 debuginfo 包（Ubunut 是 XX-dbgsym 包）

lbr
: Uses the last branch record hardware on Intel processors.
    -   较新的 Intel 处理器的新特性；

对于 `--call-graph dwarf` 方法，perf 实际是 copy the full stack from kernelspace to userspace, and then unwind
the stack using DWARF debugging info, which is relatively slow。参考：
<https://fedoraproject.org/wiki/Changes/fno-omit-frame-pointer>

如果 perf record 的是内核函数，则 perf 根据 kernel 的配置 `自动选择` CONFIG_UNWINDER_FRAME_POINTER (fp) or
CONFIG_UNWINDER_ORC (orc). 参考: [man perf-record](https://man7.org/linux/man-pages/man1/perf-record.1.html)

BFP 目前 `只支持使用 frame pointer` 来做 userspace 程序的 stack unwinding，不支持 DWARF unwinding 机制：

-   BFP 提供了 bpf_get_stackid()/bpf_get_stack() help func 来获取 userspace stack，但是它依赖于 userspace
    program 编译时开启了 frame pointer 的支持。

对于 BPF 程序，如果要 ustack() 函数正常工作，需要编译时开启 FP，即设置 -fno-omit-frame-pointer 或
--enable-frame-pointer来开启 frame pointer）来重新编译程序。

bpftrace/bcc 都不支持从额外的 debug file 中读取 DWARF debuginfo 信息的。systemtap 没有这个问题，因为它作为一个
kernle module 来加载的，无论是内核还是 User Stack，都会使用 DWARF 信息。而 BPF 依赖内核的能力，内核本身不支持
DWARF unwinder；

-   An important problem affecting bpftrace is that it `cannot generate user-space stack traces` unless the
    program being traced was `built with frame pointers`. For the vast majority of cases, that means that users
    must `recompile the software` under examination in order to instrument it.
-   bpftrace and bcc don't currently handle `detached DWARF debuginfo`, and don't even handle binaries built with
    the x64 default -fomit-frame-pointer compile flag properly.
-   <https://lwn.net/Articles/852112/>
-   <https://github.com/iovisor/bpftrace/issues/1744>

使用 `bpftrace -lv 'uprobe:/bin/bash:readline'` 来显示 readline 函数参数列表时，也是从调试符号表中解析函数名称和参数信息（参考：[dwarf_parser.cpp](https://github.com/iovisor/bpftrace/blob/master/src/dwarf_parser.cpp))。

-   如果 bpftrace 查不到调试符号表，则会报错： `No DWARF found for XX，cannot show parameter info`


## <span class="section-num">6</span> perf 采样和调用栈分析 {#perf-采样和调用栈分析}


### <span class="section-num">6.1</span> 采样：perf record {#采样-perf-record}

perf record 跟踪记录内核及应用程序的执行状态，包括调用栈：

-   -g 表示启用 call-graph（stack chain/backtrace）记录，等效于 `--call-graph=fp`  。
-   如果用户程序编译时没有开启 frame pointer，则需要安装对应的 XX-debuginfo 或 XX-dbg/XX-dbgsym 包，并指定
    `--call-graph=dwarf` ，来基于 DWARF 格式的 debuginfo 文件来提供符号表和 stack unwinding。
-   内核函数栈不支持 DWARF 的 stack unwinding，只能是 FP 或 ORC（4.14 及以后内核），perf record 自动按需选择。

<!--listend-->

```shell
#perf record -a -g -- sleep 5
[ perf record: Woken up 584 times to write data ]
Warning:
2 out of order events recorded.
[ perf record: Captured and wrote 172.619 MB perf.data (1128041 samples) ]

#du -sh perf.data
173M    perf.data
```

其他选项说明：

1.  -a： 指定采样所有 CPU 核；
2.  -p PID：指定采样指定进程；否则是所有进程；
3.  --call-graph=dwarf：使用 debuginfo 信息来对地址符号、calling stack unwinding；

生成的信息保存在 `perf.data` 中，然后通过 `perf report/script` ，就可以分析性能和调用栈。


### <span class="section-num">6.2</span> 查看函数 CPU 占用量：perf report {#查看函数-cpu-占用量-perf-report}

`perf report` 查看看哪些函数占用的 CPU 最多：

```shell
$ perf report
Samples: 24K of event 'cycles', Event count (approx.): 4868947877
  Children      Self  Command   Shared Object        Symbol
+   17.08%     0.23%  swapper   [kernel.kallsyms]    [k] do_idle
+    5.38%     5.38%  swapper   [kernel.kallsyms]    [k] intel_idle
+    4.21%     0.02%  kubelet   [kernel.kallsyms]    [k] entry_SYSCALL_64_after_hwframe
+    4.08%     0.00%  kubelet   kubelet              [.] k8s.io/kubernetes/vendor/github.com/google/...
+    4.06%     0.00%  dockerd   dockerd              [.] net/http.(*conn).serve
+    3.96%     0.00%  dockerd   dockerd              [.] net/http.serverHandler.ServeHTTP
...
```

这是一个交互式的窗口，可以选中具体函数展开查看详情。

{{< figure src="/images/perf_热点和调用栈分析/2023-08-08_15-22-08_screenshot.png" width="400" >}}


### <span class="section-num">6.3</span> 打印调用栈：perf script {#打印调用栈-perf-script}

展示采集到的事件及其调用栈：

```shell
isc-socket 15964 [003] 1912865.267505:     699832 cycles:ppp:
        ffffffffa0422fb2 _copy_from_user+0x2 ([kernel.kallsyms])
        ffffffffaFailed to open /tmp/perf-38107.map, continuing without symbols
02efbbe __x64_sys_epoll_ctl+0x4e ([kernel.kallsyms])
        ffffffffa00042e5 do_syscall_64+0x55 ([kernel.kallsyms])
        ffffffffa0a00088 entry_SYSCALL_64_after_hwframe+0x44 ([kernel.kallsyms])
            7f460786349a epoll_ctl+0xa (/usr/lib64/libc-2.17.so)

swapper     0 [005] 1912865.267508:      20833 cycles:ppp:
        ffffffffa0067b76 native_write_msr+0x6 ([kernel.kallsyms])
        ffffffffa000c987 __intel_pmu_enable_all.constprop.26+0x47 ([kernel.kallsyms])
        ffffffffa06e9492 net_rx_action+0x292 ([kernel.kallsyms])
        ffffffffa0c00108 __softirqentry_text_start+0x108 ([kernel.kallsyms])
        ffffffffa009f594 irq_exit+0xf4 ([kernel.kallsyms])
        ffffffffa0a01cb2 do_IRQ+0x52 ([kernel.kallsyms])
        ffffffffa0a009cf ret_from_intr+0x0 ([kernel.kallsyms])
            7f67d9be12e7 vfprintf+0x3df7 (/usr/lib64/libc-2.17.so)
            7f67d9c0cfa9 _IO_vsnprintf+0x79 (/usr/lib64/libc-2.17.so)
                546f4db8 [unknown] (/tmp/perf-38107.map)

sqlonline_worke 38771 [006] 1912865.267542:          1 cycles:ppp:
        ffffffffa0067b76 native_write_msr+0x6 ([kernel.kallsyms])
        ffffffffa000c987 __intel_pmu_enable_all.constprop.26+0x47 ([kernel.kallsyms])
        ffffffffa01d27d3 event_function+0x83 ([kernel.kallsyms])
        ffffffffa01d3e69 remote_function+0x39 ([kernel.kallsyms])
        ffffffffa0139f50 flush_smp_call_function_queue+0x70 ([kernel.kallsyms])
        ffffffffa0a023b4 smp_call_function_single_interrupt+0x34 ([kernel.kallsyms])
        ffffffffa0a01b5f call_function_single_interrupt+0xf ([kernel.kallsyms])
        ffffffffc0362a76 ipt_do_table+0x2a6 ([kernel.kallsyms])
        ffffffffa073abbd nf_hook_slow+0x3d ([kernel.kallsyms])
        ffffffffa07471c3 ip_rcv+0xa3 ([kernel.kallsyms])
        ffffffffa06e8b00 __netif_receive_skb_one_core+0x50 ([kernel.kallsyms])
        ffffffffa06e7d02 netif_receive_skb_internal+0x42 ([kernel.kallsyms])
        ffffffffa06e9cf5 napi_gro_receive+0xb5 ([kernel.kallsyms])
        ffffffffc051da6b ixgbe_clean_rx_irq+0x49b ([kernel.kallsyms])
        ffffffffc051ef1b ixgbe_poll+0x2ab ([kernel.kallsyms])
        ffffffffa06e9492 net_rx_action+0x292 ([kernel.kallsyms])
        ffffffffa0c00108 __softirqentry_text_start+0x108 ([kernel.kallsyms])
        ffffffffa009f594 irq_exit+0xf4 ([kernel.kallsyms])
        ffffffffa0a01cb2 do_IRQ+0x52 ([kernel.kallsyms])
        ffffffffa0a009cf ret_from_intr+0x0 ([kernel.kallsyms])
            7f67d9be12e7 vfprintf+0x3df7 (/usr/lib64/libc-2.17.so)
            7f67d9c0cfa9 _IO_vsnprintf+0x79 (/usr/lib64/libc-2.17.so)
                546f4db8 [unknown] (/tmp/perf-38107.map)
```

可以指定一些参数，如 `perf script -c bash` 来只显示 bash 命令的调用栈。


### <span class="section-num">6.4</span> 生成火焰图：perf script | ... &gt; result.svg {#生成火焰图-perf-script-dot-dot-dot-result-dot-svg}

将 perf script 的输出重定向到 perl 脚本做进一步处理，就得到了火焰图：

```text
perf script | ./stackcollapse-perf.pl | ./flamegraph.pl > result.svg
```

注意：

1.  一般是特定进程的火焰图，所以在 perf record 时需要通过 -p 来指定进程 PID；
2.  生成火焰图依赖 debuginfo 数据。需要采样的进程二进制以及依赖库如 libc，包含 .debug_XX 调试符号表或系统安装有对应的 debuginfo 包。


## <span class="section-num">7</span> perf probe {#perf-probe}

示例函数：

```c
root@lima-ebpf-dev:~# cat test.c
#include <stdio.h>
#include <unistd.h>

void func_d() {
                int msec=1;
                printf("%s","Hello world from D\n");
                usleep(10000*msec);
}
void func_c() {
                printf("%s","Hello from C\n");
                func_d();
}
void func_b() {
                printf("%s","Hello from B\n");
        func_c();
}
void func_a() {
                printf("%s","Hello from A\n");
                func_b();
}
int main() {
        func_a();
}
root@lima-ebpf-dev:~#
```

编译和 perf probe：

```shell
root@lima-ebpf-dev:~# gcc test.c -g -o test # 保存 FP

root@lima-ebpf-dev:~# perf probe -x ./hello func_d
Added new event:
  probe_hello:func_d   (on func_d in /root/hello)

You can now use it in all perf tools, such as:

        perf record -e probe_hello:func_d -aR sleep 1

root@lima-ebpf-dev:~# perf record -e probe_hello:func_d -aR -g ./hello  # -g 默认使用 FP，也可以指定 -g --call-graph dwarf
Hello from A
Hello from B
Hello from C
Hello world from D
[ perf record: Woken up 1 times to write data ]
[ perf record: Captured and wrote 0.162 MB perf.data (1 samples) ]
root@lima-ebpf-dev:~#

root@lima-ebpf-dev:~# perf script
hello 14239 [002]  5516.587824: probe_hello:func_d: (561112f12169)
            561112f12169 func_d+0x0 (/root/hello)
            561112f121e3 func_b+0x21 (/root/hello)
            561112f12207 func_a+0x21 (/root/hello)
            561112f1221c main+0x12 (/root/hello)
            7fe3e67e1d90 __libc_start_call_main+0x80 (/usr/lib/x86_64-linux-gnu/libc.so.6)
```

参考：

1.  [Practical Linux tracing ( Part 1/5) : symbols, debug symbols and stack unwinding](https://medium.com/coccoc-engineering-blog/things-you-should-know-to-begin-playing-with-linux-tracing-tools-part-i-x-225aae1aaf13)


## <span class="section-num">8</span> bpftrace 跟踪内核函数调用栈 {#bpftrace-跟踪内核函数调用栈}

可以使用 `bpftrace -l` 来查询支持的 kprobe 内核函数：

```shell
#bpftrace -l 'kprobe:*nf_conn*'|head
kprobe:nf_conntrack_destroy
kprobe:nf_conntrack_double_unlock
kprobe:__nf_conntrack_hash_insert
kprobe:nf_conntrack_attach
kprobe:nf_conntrack_lock
kprobe:nf_conntrack_free
kprobe:nf_conntrack_alter_reply
kprobe:nf_conntrack_double_lock.isra.32
kprobe:nf_conntrack_hash_check_insert
kprobe:nf_conntrack_tuple_taken
```

然后使用 -e 来指定该 event，同时打印 kstack：

```shell
# bpftrace -e 'kprobe:nf_conntrack_in {printf("%s\n", kstack); }'
        nf_conntrack_in+1
        nf_hook_slow+61
        __ip_local_out+214
        ip_local_out+23
        ip_send_skb+21
        udp_send_skb.isra.43+277
        udp_sendmsg+1544
        sock_sendmsg+48
        ___sys_sendmsg+688
        __sys_sendmsg+99
        do_syscall_64+85
        entry_SYSCALL_64_after_hwframe+68
```


## <span class="section-num">9</span> bpftrace 跟踪用户程序执行 {#bpftrace-跟踪用户程序执行}

注：bpftrace 不支持基于 DWARF 的 stack unwinding，需要用户程序编译时生成 frame pointer。

1.  执行使用 bpftrace 执行程序；

<!--listend-->

```c
root@lima-ebpf-dev:~# cat test.c
#include <stdio.h>
#include <unistd.h>

void func_d() {
                int msec=1;
                printf("%s","Hello world from D\n");
                usleep(10000*msec);
}
void func_c() {
                printf("%s","Hello from C\n");
                func_d();
}
void func_b() {
                printf("%s","Hello from B\n");
        func_c();
}
void func_a() {
                printf("%s","Hello from A\n");
                func_b();
}
int main() {
        func_a();
}
```

```shell
# 没有指定 -O 优化选项，所以开启 FP
root@lima-ebpf-dev:~# gcc  test.c -o hello

# 确认 gcc 在函数调用的开头添加保存 FP 的指令。
root@lima-ebpf-dev:~# objdump -S hello |grep -A 4 func_c
000000000000119e <func_c>:
    119e:       f3 0f 1e fa             endbr64
    11a2:       55                      push   %rbp  # 保存 FP
    11a3:       48 89 e5                mov    %rsp,%rbp
    11a6:       48 8d 05 6a 0e 00 00    lea    0xe6a(%rip),%rax        # 2017 <_IO_stdin_used+0x17>
--
    11de:       e8 bb ff ff ff          call   119e <func_c>
    11e3:       90                      nop
    11e4:       5d                      pop    %rbp
    11e5:       c3                      ret

# 打印调用 func_c 的 user call stack
root@lima-ebpf-dev:~# bpftrace -e 'uprobe:./hello:func_c {printf("%s",ustack)}' -c ./hello
Attaching 1 probe...
Hello from A
Hello from B
Hello from C
Hello world from D

        func_c+0
        func_a+33
        main+18
        __libc_start_call_main+128
```

1.  使用 pid 追踪正在运行的程序；
    -   二进制程序需要支持 FP，才能进行 stack unwinding。
    -   如果加 -p 则只 probe 特定进程的函数调用, 否则是系统范围内执行该二进制的函数.

<!--listend-->

```shell
root@lima-ebpf-dev:~# apt install bash-dbgsym bash-static-dbgsym
root@lima-ebpf-dev:~# bpftrace -e 'uprobe:/usr/bin/bash:readline {printf("%s", ustack)}' # -p 12446
```


## <span class="section-num">10</span> bpftrace 跟踪容器方式部署的应用 {#bpftrace-跟踪容器方式部署的应用}

如果应用程序跑在容器内，在宿主机用 bpftrace 跟踪时，需要一些额外信息。


### <span class="section-num">10.1</span> 指定目标文件的绝对路径 {#指定目标文件的绝对路径}

目标文件在宿主机上的绝对路径。

例如，如果想跟踪 cilium-agent 进程（本身是用 docker 容器部署的），首先需要找到 cilium-agent 文件在宿主机上的绝对路径，可以通过 container ID 或 name 找：

```shell
# Check cilium-agent container
$ docker ps | grep cilium-agent
0eb2e76384b3        cilium:test   "/usr/bin/cilium-agent ..."   4 hours ago    Up 4 hours   cilium-agent

# Find the merged path for cilium-agent container
$ docker inspect --format "{{.GraphDriver.Data.MergedDir}}" 0eb2e76384b3
/var/lib/docker/overlay2/a17f868d/merged # a17f868d.. is shortened for better viewing

# The object file we are going to trace
$ ls -ahl /var/lib/docker/overlay2/a17f868d/merged/usr/bin/cilium-agent
-rwxr-xr-x 1 root root 86M /var/lib/docker/overlay2/a17f868d/merged/usr/bin/cilium-agent
```

也可以暴力一点直接 find：

```shell
(node) $ find /var/lib/docker/overlay2/ -name cilium-agent
/var/lib/docker/overlay2/a17f868d/merged/usr/bin/cilium-agent
```

然后再指定绝对路径 uprobe：

-   go 函数需要包含完整路径,如 "github.com/cilium/cilium/pkg/endpoint.(\*Endpoint).regenerate"

<!--listend-->

```shell
(node) $ bpftrace -e 'uprobe:/var/lib/docker/overlay2/a17f868d/merged/usr/bin/cilium-agent:"github.com/cilium/cilium/pkg/endpoint.(*Endpoint).regenerate" {printf("%s\n", ustack); }'
Attaching 1 probe...

        github.com/cilium/cilium/pkg/endpoint.(*Endpoint).regenerate+0
        github.com/cilium/cilium/pkg/eventqueue.(*EventQueue).run.func1+363
        sync.(*Once).doSlow+236
        github.com/cilium/cilium/pkg/eventqueue.(*EventQueue).run+101
        runtime.goexit+1
```

可以使用 nm 或者 bptrace 命令来查看 go 二进制中可以 tracing 的符号（函数）列表：

```shell
$ nm cilium-agent
000000000427d1d0 B bufio.ErrBufferFull
000000000427d1e0 B bufio.ErrFinalToken
0000000001d3e940 T type..hash.github.com/cilium/cilium/pkg/k8s.ServiceID
0000000001f32300 T type..hash.github.com/cilium/cilium/pkg/node/types.Identity
0000000001d05620 T type..hash.github.com/cilium/cilium/pkg/policy/api.FQDNSelector
0000000001d05e80 T type..hash.github.com/cilium/cilium/pkg/policy.PortProto
...

root@lima-ebpf-dev:# bpftrace -l 'uprobe:./exec:*'|tail
uprobe:./exec:vendor/golang.org/x/text/unicode/norm.lookupInfoNFC
uprobe:./exec:vendor/golang.org/x/text/unicode/norm.lookupInfoNFKC
uprobe:./exec:vendor/golang.org/x/text/unicode/norm.nextCGJCompose
uprobe:./exec:vendor/golang.org/x/text/unicode/norm.nextCGJDecompose
uprobe:./exec:vendor/golang.org/x/text/unicode/norm.nextComposed
uprobe:./exec:vendor/golang.org/x/text/unicode/norm.nextDecomposed
uprobe:./exec:vendor/golang.org/x/text/unicode/norm.nextDone
uprobe:./exec:vendor/golang.org/x/text/unicode/norm.nextHangul
uprobe:./exec:vendor/golang.org/x/text/unicode/norm.nextMulti
uprobe:./exec:vendor/golang.org/x/text/unicode/norm.nextMultiNorm
```


### <span class="section-num">10.2</span> 指定目标进程 PID /proc/&lt;PID&gt; {#指定目标进程-pid-proc-pid}

-   二进制路径为 `/proc/<pid>/root` 下的路径.

<!--listend-->

```shell
$ sudo docker inspect -f '{{.State.Pid}}' cilium-agent
109997
 (node) $ bpftrace -e 'uprobe:/proc/109997/root/usr/bin/cilium-agent:"github.com/cilium/cilium/pkg/endpoint.(*Endpoint).regenerate" {printf("%s\n", ustack); }'
```


### <span class="section-num">10.3</span> 指定目标进程 PID -p &lt;PID&gt; {#指定目标进程-pid-p-pid}

-   二进制路径为容器内路径地址.

<!--listend-->

```shell
(node) $ bpftrace -p 109997 -e 'uprobe:/usr/bin/cilium-agent:"github.com/cilium/cilium/pkg/endpoint.(*Endpoint).rege
```


## <span class="section-num">11</span> 参考 {#参考}

1.  [Linux tracing/profiling 基础：符号表、调用栈、perf/bpftrace 示例等（2022）](http://arthurchiao.art/blog/linux-tracing-basis-zh/)
