---
title: "Function Stack Unwinding"
author: ["opsnull"]
date: 2023-08-07T00:00:00+08:00
lastmod: 2023-08-07T22:21:29+08:00
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
2.  DWARF CFI（Call Frame Information）；
3.  ORC：4.14及以后版本内核专用，简化版的 DWARF CFI；
4.  LBR：新的 Intel CPU支持；


## <span class="section-num">1</span> FP {#fp}

frame pointer 是通过一个特定的 CPU 寄存器(rbp)来保存栈指针，由编译器在函数调用和退出时添加额外的指令来 save、
setup 和 restore 该 CPU 寄存器中保存的 frame pointer。

-   基本原理：函数调用时，gcc 会将函数参数 args 、函数自动变量、函数返回地址、当前栈指针（ rsp 寄存器中保存）push
    当前frame stack，这样通过 rbp获得上一个栈指针后，通过固定偏移获得当前栈上保存的函数返回地址，再使用返回地址来查找 binary 中的符号表（.symtab) 来获得对应的函数名称。

frame pointer 是用户空间程序通用的stack unwinding机制，但是由于性能和开销问题，从GCC 4.6开始，各大发行版
(Centos7/Ubuntu 20.04)发布的软件包`都默认关闭了 frame pointer`，转而使用 DWARF 信息来做stack unwinding。这会导致 bcc/bpftrace等依赖 FP 来做uprobe/usdt的 stack unwinding 功能不可用，所以关闭了 FP 后，它们不能正常打印
ustack；

Fedora 38 开始[默认在编译软件包时开启-fno-omit-frame-pointer](https://pagure.io/fesco/issue/2923)编译器参数：

Golang [从1.7开始](https://github.com/golang/go/issues/15840)对 amd64 提供frame pointer的支持。（[internal-abi.md](https://go.googlesource.com/go/+/refs/heads/dev.regabi/src/cmd/compile/internal-abi.md))


## <span class="section-num">2</span> GCC 优化和 FP {#gcc-优化和-fp}

从GCC 4.6开始，[只要开启了优化(从-O/-O1开始), 就会关闭 frame pointer](https://gcc.gnu.org/bugzilla//show_bug.cgi?id=100811)：

-O/-O1
: Optimize. Optimizing compilation takes somewhat more time, and a lot more memory for a large
    function. With ‘ -O ’, the compiler `tries to reduce code size and execution time`, without performing any
    optimizations that take a great deal of compilation time.

-O2
: Optimize even more. GCC performs nearly `all supported optimizations` that do not involve a space-speed
    tradeoff. As compared to ‘ -O ’, this option increases both compilation time and the performance of the
    generated code.

-O3
: Optimize yet more. ‘ -O3 ’turns on all optimizations specified by‘ -O2 ’ and also turns on the
    following optimization flags:

-Os
: Optimize for size. ‘ -Os ’enables all‘ -O2 ’ optimizations except those that often increase code size:

-O0
: Reduce compilation time and make debugging produce the expected results. `This is the default`.

如果要开启 FP ，则在使用 gcc时：

1.  不开启优化，如不指定任何 -O 选项或指定 -O0；
2.  或者明确指定参数：-fno-omit-frame-pointer 或 --enable-frame-pointer;

-fomit-frame-pointer 解释：Omit the frame pointer in functions that don’t need one. This avoids the
instructions to save, set up and restore the frame pointer; on many targets it also makes an extra register
available.  On some targets this flag has no effect because the standard calling sequence always uses a frame
pointer, so it cannot be omitted.  Note that `‘ -fno-omit-frame-pointer ’` doesn’t guarantee the frame pointer is
used in all functions. Several targets always omit the frame pointer in leaf functions. `Enabled by default at
‘ -O ’ and higher`.

----

For x86-64, the ABI (PDF) `encourages the absence of a frame pointer`. The rationale is more or less "we have
DWARF now, so it's not necessary for debugging or exception unwinding; if we make it optional from day one,
then no software will come to depend on its existence."

x86-64 does have more registers than x86-32, but it `still doesn't have enough`. Freeing up more general-purpose
registers is always a Good Thing from a compiler's point of view. The operations that require a stack crawl
are slower, yes, but they are rare events, so it's a good tradeoff for shaving a few cycles off every
subroutine call plus fewer stack spills.


## <span class="section-num">3</span> DWARF CFI {#dwarf-cfi}

frame pinter 并不是运行程序所必须的, 而只在debugger unwinding frame时才需要。而编译器了解所有 func stack
frame 的大小和分配情况, 所以在编译时可以将func stack frame信息写入 elf 的 .debug_XX Sections 或 debuginfo 文件, 后续 debugger 通过读取这些内容来判断函数调用关系，这就出现了DWARF CFI规范.

[DWARF CFI（DWARF 3 Spec）](https://refspecs.linuxfoundation.org/LSB_3.0.0/LSB-PDA/LSB-PDA.junk/dwarfext.html) 是一种在 EFL 文件的`.debug_frame` Section中保存 unwind 信息的规范，通过解析这些信息，可以 unwind stack：

-   通过编译时添加 -g 选项来生成；
-   .debug_frame 不需要加载到内存，可以位于二进制外的单独 debuginfo 文件中；
-   Linus 反对内核自身使用 DWARF 来做backtrace unwind的邮件：<https://lkml.org/lkml/2012/2/10/356%EF%BC%8C%E6%89%80%E4%BB%A5> DWARF
    unwinder 只在用户空间程序使用；
-   使用readelf -w XX来查看.debug_frame的内容；

在编译的二进制不支持stack frame pointer（ gcc 开启优化有就关闭 FP，故各大发行版提供的软件包一般都不支持 FP）时，
gdb/perf 使用 debuginfo （DWARF格式，一般使用GNU libunwind包，或 elfutils 提供的 libdw ）来进行 stack
unwinding/单步调试/内存地址和源文件对应关系。

在 CFI 的基础上，又提出了[exception handler framework(.eh_frame)](https://refspecs.linuxfoundation.org/LSB_3.0.0/LSB-Core-generic/LSB-Core-generic/ehframechpt.html),用来解决各种语言的结构对 reliable unwinding
的需求。`.eh_frame`是 .debug_frame 的子集，遵从DWARF CFI Extensions规范，`是 unwinding stack专用的特性` （而
.debug_frame 功能更通用)：

-   .eh_frame 会被 gcc 默认生成，不会被 strip ，会被加载到内存（而 .debug_frame等是会被 strip，默认不会被加载到内存）。
-   使用readelf -Wwf XX来查看.eh_frame section的内容；

C++ 在任意位置可能触发的异常也是通过DWARF .eh_frame table（也称 unwind table）来实现Exception Stack展示的。由于异常处理需要编译器插入额外的异常处理代码和调试信息，会导致结果二进制文件变大，所以 gcc 支持使用-fno-exceptions 和 -fno-unwind-tables 来关闭C++的 Exception 功能。

对于 DWARF 的解析，主要有两个 library：

1.  `BFD (libbfd)` is used by the `GNU binutils`, including objdump which played a star role in this article, ld
    (the GNU linker) and as (the GNU assembler).
2.  `libdwarf` - which together with its big brother libelf are used for the tools on Solaris and FreeBSD
    operating systems.

对于stack unwinding支持，主要有两个库：`GNU libunwind`和 `elfutils 的libdw.so`，它们都支持 FP 和 DWARF
unwinding：

1.  libundwind : 使用 debuginfo （DWARF格式）需要安装 libunwind 库，这样在进程二进制没有编译支持 stack frame
    pointer 时，基于 DWARF 来进行 stack unwinding：
    -   需要libunwind 1.1及以上版本，且编译时配置了 --enable-debug-frame ， CentOS 7均支持；
        ```shell
             yum install libunwind libunwind-debuginfo -y
        ```
2.  libdw.so: elfutils 项目提供的libdw.so库也支持基于 DWARF 的 stack unwind 且性能比 libunwind 好一些。当前
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


## <span class="section-num">4</span> Kernel ORC {#kernel-orc}

linus 没有接收使用 DWARF 作为 kernel 的 frame unwinder 机制，还是坚守frame pointer机制。但是从 Kernel v4.14
开始（CentOS 8 开始），内核开始使用ORC unwinder机制来作为 DWARF 的简化实现。

-   [x86: ORC unwinder (previously undwarf)](https://lwn.net/Articles/727553/)
-   [Unwinding a Stack by Hand with Frame Pointers and ORC](https://blogs.oracle.com/linux/post/unwinding-stack-frame-pointers-and-orc)

内核编译选项：

1.  CONFIG_FRAME_POINTER：编译开启了frame pointers的内核
    -   支持的版本： 2.6.9–2.6.39, 3.0–3.19, 4.0–4.20, 5.0–5.19, 6.0–6.4, 6.5-rc+HEAD
2.  CONFIG_UNWINDER_ORC: 编译开启了ORC unwiner的内核，用来 unwinding kernel stack traces。
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

perf record 支持捕获函数调用栈, 对于`用户空间程序`，可以使用 --call-grah 选项来指定unwinding stack的方法：

```text
perf record --call-graph method command
```

fp
: Uses the `frame pointer method`. Depending on compiler optimization, such as with binaries built with the
    GCC option --fomit-frame-pointer, this may not be able to unwind the stack.
    -   对于当前发行版自带的用户空间程序，由于当前发行版默认都开了优化，也就是默认开启了 --fomit-frame-pointer ，所以该方法可能不适用；（但是如果是用户自己编译的程序，则可以设置 -fno-omit-frame-pointer或
        --enable-frame-pointer 来开启 frame pointer）；

dwarf
: Uses `DWARF Call Frame Information` to unwind the stack.
    -   需要安装了可执行程序的 debuginfo 包（Ubunut 是 XX-dbgsym 包）

lbr
: Uses the last branch record hardware on Intel processors.
    -   较新的 Intel 处理器的新特性；

对于`--call-graph dwarf`方法，perf 实际是 copy the full stack from kernelspace to userspace, and then unwind
the stack using DWARF debugging info, which is relatively slow。参考：
<https://fedoraproject.org/wiki/Changes/fno-omit-frame-pointer>

如果perf record的是内核函数，则 perf 根据 kernel 的配置 `自动选择` CONFIG_UNWINDER_FRAME_POINTER (fp) or
CONFIG_UNWINDER_ORC (orc). 参考: [man perf-record](https://man7.org/linux/man-pages/man1/perf-record.1.html)

BFP 目前`只支持使用 frame pointer`来做 userspace 程序的stack unwinding，不支持 DWARF unwinding机制：

-   BFP 提供了bpf_get_stackid()/bpf_get_stack() help func来获取 userspace stack，但是它依赖于 userspace
    program 编译时开启了frame pointer的支持。

对于 BPF 程序，如果要ustack()函数正常工作，需要编译时开启 FP ，即设置 -fno-omit-frame-pointer或
--enable-frame-pointer来开启 frame pointer）来重新编译程序。

bpftrace/bcc 都不支持从额外的debug file中读取DWARF debuginfo信息的。systemtap 没有这个问题，因为它作为一个
kernle module 来加载的，无论是内核还是User Stack，都会使用 DWARF信息。而 BPF 依赖内核的能力，内核本身不支持
DWARF unwinder；

-   An important problem affecting bpftrace is that it `cannot generate user-space stack traces` unless the
    program being traced was `built with frame pointers`. For the vast majority of cases, that means that users
    must `recompile the software` under examination in order to instrument it.
-   bpftrace and bcc don't currently handle `detached DWARF debuginfo`, and don't even handle binaries built with
    the x64 default -fomit-frame-pointer compile flag properly.
-   <https://lwn.net/Articles/852112/>
-   <https://github.com/iovisor/bpftrace/issues/1744>

使用`bpftrace -lv 'uprobe:/bin/bash:readline'`来显示 readline 函数参数列表时，也是从调试符号表中解析函数名称和参数信息（参考：[dwarf_parser.cpp](https://github.com/iovisor/bpftrace/blob/master/src/dwarf_parser.cpp))。

-   如果 bpftrace 查不到调试符号表，则会报错： `No DWARF found for XX，cannot show parameter info`
