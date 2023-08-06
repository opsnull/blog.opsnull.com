---
title: "objdump"
author: ["opsnull"]
date: 2023-08-06T00:00:00+08:00
lastmod: 2023-08-06T22:07:45+08:00
tags: ["linux", "elf", "debug", "tools"]
categories: ["debug", "tools"]
draft: false
series: ["elf-debug"]
series_order: 4
---

介绍 objdump 命令的功能用法。

<!--more-->

readelf 和 objdump 相比：

1.  objdump 可以对二进制文件根据调试符号表进行反汇编，但是 readelf 不行；
2.  两者都会使用 build-id 和 gnu debuglink 机制从`/usr/lib/debug`查找当前二进制的 debuginfo 文件，然后显示其内容；

通用选项：

1.  `-w/--wide` ：可以使输出宽度超过 80 字符。
2.  `-j, --section=NAME` ：只显示执行 SECTION 的内容。


## <span class="section-num">1</span> 显示文件格式 {#显示文件格式}

objdump -a 显示文件格式（elf 也是一种归档 archive 文件格式）：

```shell
root@lima-ebpf-dev:~# objdump -w -a hello

hello:     file format elf64-x86-64
hello
```


## <span class="section-num">2</span> 显示 ELF header {#显示-elf-header}

objdump -f 显示elf header的简略概要：

```shell
root@lima-ebpf-dev:~# objdump -w -f hello

hello:     file format elf64-x86-64
architecture: i386:x86-64, flags 0x00000150:
HAS_SYMS, DYNAMIC, D_PAGED
start address 0x0000000000001060
```


## <span class="section-num">3</span> 显示 Program headers {#显示-program-headers}

objdump -p 显示 elf 文件中的 private-headers 列表：

-   包括：Program headers，Dynamic Section Headers

<!--listend-->

```shell
root@lima-ebpf-dev:~# objdump -w -p hello

hello:     file format elf64-x86-64

Program Header:
    PHDR off    0x0000000000000040 vaddr 0x0000000000000040 paddr 0x0000000000000040 align 2**3
         filesz 0x00000000000002d8 memsz 0x00000000000002d8 flags r--
  INTERP off    0x0000000000000318 vaddr 0x0000000000000318 paddr 0x0000000000000318 align 2**0
         filesz 0x000000000000001c memsz 0x000000000000001c flags r--
    LOAD off    0x0000000000000000 vaddr 0x0000000000000000 paddr 0x0000000000000000 align 2**12
         filesz 0x0000000000000628 memsz 0x0000000000000628 flags r--
    LOAD off    0x0000000000001000 vaddr 0x0000000000001000 paddr 0x0000000000001000 align 2**12
         filesz 0x0000000000000185 memsz 0x0000000000000185 flags r-x
    LOAD off    0x0000000000002000 vaddr 0x0000000000002000 paddr 0x0000000000002000 align 2**12
         filesz 0x0000000000000114 memsz 0x0000000000000114 flags r--
    LOAD off    0x0000000000002db8 vaddr 0x0000000000003db8 paddr 0x0000000000003db8 align 2**12
         filesz 0x0000000000000258 memsz 0x0000000000000260 flags rw-
 DYNAMIC off    0x0000000000002dc8 vaddr 0x0000000000003dc8 paddr 0x0000000000003dc8 align 2**3
         filesz 0x00000000000001f0 memsz 0x00000000000001f0 flags rw-
    NOTE off    0x0000000000000338 vaddr 0x0000000000000338 paddr 0x0000000000000338 align 2**3
         filesz 0x0000000000000030 memsz 0x0000000000000030 flags r--
    NOTE off    0x0000000000000368 vaddr 0x0000000000000368 paddr 0x0000000000000368 align 2**2
         filesz 0x0000000000000044 memsz 0x0000000000000044 flags r--
0x6474e553 off    0x0000000000000338 vaddr 0x0000000000000338 paddr 0x0000000000000338 align 2**3
         filesz 0x0000000000000030 memsz 0x0000000000000030 flags r--
EH_FRAME off    0x000000000000200c vaddr 0x000000000000200c paddr 0x000000000000200c align 2**2
         filesz 0x000000000000003c memsz 0x000000000000003c flags r--
   STACK off    0x0000000000000000 vaddr 0x0000000000000000 paddr 0x0000000000000000 align 2**4
         filesz 0x0000000000000000 memsz 0x0000000000000000 flags rw-
   RELRO off    0x0000000000002db8 vaddr 0x0000000000003db8 paddr 0x0000000000003db8 align 2**0
         filesz 0x0000000000000248 memsz 0x0000000000000248 flags r--

Dynamic Section:
  NEEDED               libc.so.6
  INIT                 0x0000000000001000
  FINI                 0x0000000000001178
  INIT_ARRAY           0x0000000000003db8
  INIT_ARRAYSZ         0x0000000000000008
  FINI_ARRAY           0x0000000000003dc0
  FINI_ARRAYSZ         0x0000000000000008
  GNU_HASH             0x00000000000003b0
  STRTAB               0x0000000000000480
  SYMTAB               0x00000000000003d8
  STRSZ                0x000000000000008d
  SYMENT               0x0000000000000018
  DEBUG                0x0000000000000000
  PLTGOT               0x0000000000003fb8
  PLTRELSZ             0x0000000000000018
  PLTREL               0x0000000000000007
  JMPREL               0x0000000000000610
  RELA                 0x0000000000000550
  RELASZ               0x00000000000000c0
  RELAENT              0x0000000000000018
  FLAGS                0x0000000000000008
  FLAGS_1              0x0000000008000001
  VERNEED              0x0000000000000520
  VERNEEDNUM           0x0000000000000001
  VERSYM               0x000000000000050e
  RELACOUNT            0x0000000000000003

Version References:
  required from libc.so.6:
    0x09691a75 0x00 03 GLIBC_2.2.5
    0x069691b4 0x00 02 GLIBC_2.34
```


## <span class="section-num">4</span> 显示 Section headers {#显示-section-headers}

objdump -h 显示Section Header列表：

```shell
root@lima-ebpf-dev:~# objdump -h hello

hello:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .interp       0000001c  0000000000000318  0000000000000318  00000318  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  1 .note.gnu.property 00000030  0000000000000338  0000000000000338  00000338  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .note.gnu.build-id 00000024  0000000000000368  0000000000000368  00000368  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .note.ABI-tag 00000020  000000000000038c  000000000000038c  0000038c  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .gnu.hash     00000024  00000000000003b0  00000000000003b0  000003b0  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  5 .dynsym       000000a8  00000000000003d8  00000000000003d8  000003d8  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  6 .dynstr       0000008d  0000000000000480  0000000000000480  00000480  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  7 .gnu.version  0000000e  000000000000050e  000000000000050e  0000050e  2**1
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  8 .gnu.version_r 00000030  0000000000000520  0000000000000520  00000520  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  9 .rela.dyn     000000c0  0000000000000550  0000000000000550  00000550  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 10 .rela.plt     00000018  0000000000000610  0000000000000610  00000610  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 11 .init         0000001b  0000000000001000  0000000000001000  00001000  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 12 .plt          00000020  0000000000001020  0000000000001020  00001020  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 13 .plt.got      00000010  0000000000001040  0000000000001040  00001040  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 14 .plt.sec      00000010  0000000000001050  0000000000001050  00001050  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 15 .text         00000117  0000000000001060  0000000000001060  00001060  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 16 .fini         0000000d  0000000000001178  0000000000001178  00001178  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 17 .rodata       0000000b  0000000000002000  0000000000002000  00002000  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 18 .eh_frame_hdr 0000003c  000000000000200c  000000000000200c  0000200c  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 19 .eh_frame     000000cc  0000000000002048  0000000000002048  00002048  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 20 .init_array   00000008  0000000000003db8  0000000000003db8  00002db8  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 21 .fini_array   00000008  0000000000003dc0  0000000000003dc0  00002dc0  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 22 .dynamic      000001f0  0000000000003dc8  0000000000003dc8  00002dc8  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 23 .got          00000048  0000000000003fb8  0000000000003fb8  00002fb8  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 24 .data         00000010  0000000000004000  0000000000004000  00003000  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 25 .bss          00000008  0000000000004010  0000000000004010  00003010  2**0
                  ALLOC
 26 .comment      0000002b  0000000000000000  0000000000000000  00003010  2**0
                  CONTENTS, READONLY
 27 .debug_aranges 00000030  0000000000000000  0000000000000000  0000303b  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
 28 .debug_info   000000a6  0000000000000000  0000000000000000  0000306b  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
 29 .debug_abbrev 0000005e  0000000000000000  0000000000000000  00003111  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
 30 .debug_line   0000005f  0000000000000000  0000000000000000  0000316f  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
 31 .debug_str    000000df  0000000000000000  0000000000000000  000031ce  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
 32 .debug_line_str 00000011  0000000000000000  0000000000000000  000032ad  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
root@lima-ebpf-dev:~#
```


## <span class="section-num">5</span> 显示所有 Headers {#显示所有-headers}

objdump -x 显示所有 header 列表：

```shell
root@lima-ebpf-dev:~# objdump -w -x hello

hello:     file format elf64-x86-64
hello
architecture: i386:x86-64, flags 0x00000150:
HAS_SYMS, DYNAMIC, D_PAGED
start address 0x0000000000001060

Program Header:
    PHDR off    0x0000000000000040 vaddr 0x0000000000000040 paddr 0x0000000000000040 align 2**3
         filesz 0x00000000000002d8 memsz 0x00000000000002d8 flags r--
  INTERP off    0x0000000000000318 vaddr 0x0000000000000318 paddr 0x0000000000000318 align 2**0
         filesz 0x000000000000001c memsz 0x000000000000001c flags r--
    LOAD off    0x0000000000000000 vaddr 0x0000000000000000 paddr 0x0000000000000000 align 2**12
         filesz 0x0000000000000628 memsz 0x0000000000000628 flags r--
    LOAD off    0x0000000000001000 vaddr 0x0000000000001000 paddr 0x0000000000001000 align 2**12
         filesz 0x0000000000000185 memsz 0x0000000000000185 flags r-x
    LOAD off    0x0000000000002000 vaddr 0x0000000000002000 paddr 0x0000000000002000 align 2**12
         filesz 0x0000000000000114 memsz 0x0000000000000114 flags r--
    LOAD off    0x0000000000002db8 vaddr 0x0000000000003db8 paddr 0x0000000000003db8 align 2**12
         filesz 0x0000000000000258 memsz 0x0000000000000260 flags rw-
 DYNAMIC off    0x0000000000002dc8 vaddr 0x0000000000003dc8 paddr 0x0000000000003dc8 align 2**3
         filesz 0x00000000000001f0 memsz 0x00000000000001f0 flags rw-
    NOTE off    0x0000000000000338 vaddr 0x0000000000000338 paddr 0x0000000000000338 align 2**3
         filesz 0x0000000000000030 memsz 0x0000000000000030 flags r--
    NOTE off    0x0000000000000368 vaddr 0x0000000000000368 paddr 0x0000000000000368 align 2**2
         filesz 0x0000000000000044 memsz 0x0000000000000044 flags r--
0x6474e553 off    0x0000000000000338 vaddr 0x0000000000000338 paddr 0x0000000000000338 align 2**3
         filesz 0x0000000000000030 memsz 0x0000000000000030 flags r--
EH_FRAME off    0x000000000000200c vaddr 0x000000000000200c paddr 0x000000000000200c align 2**2
         filesz 0x000000000000003c memsz 0x000000000000003c flags r--
   STACK off    0x0000000000000000 vaddr 0x0000000000000000 paddr 0x0000000000000000 align 2**4
         filesz 0x0000000000000000 memsz 0x0000000000000000 flags rw-
   RELRO off    0x0000000000002db8 vaddr 0x0000000000003db8 paddr 0x0000000000003db8 align 2**0
         filesz 0x0000000000000248 memsz 0x0000000000000248 flags r--

Dynamic Section:
  NEEDED               libc.so.6
  INIT                 0x0000000000001000
  FINI                 0x0000000000001178
  INIT_ARRAY           0x0000000000003db8
  INIT_ARRAYSZ         0x0000000000000008
  FINI_ARRAY           0x0000000000003dc0
  FINI_ARRAYSZ         0x0000000000000008
  GNU_HASH             0x00000000000003b0
  STRTAB               0x0000000000000480
  SYMTAB               0x00000000000003d8
  STRSZ                0x000000000000008d
  SYMENT               0x0000000000000018
  DEBUG                0x0000000000000000
  PLTGOT               0x0000000000003fb8
  PLTRELSZ             0x0000000000000018
  PLTREL               0x0000000000000007
  JMPREL               0x0000000000000610
  RELA                 0x0000000000000550
  RELASZ               0x00000000000000c0
  RELAENT              0x0000000000000018
  FLAGS                0x0000000000000008
  FLAGS_1              0x0000000008000001
  VERNEED              0x0000000000000520
  VERNEEDNUM           0x0000000000000001
  VERSYM               0x000000000000050e
  RELACOUNT            0x0000000000000003

Version References:
  required from libc.so.6:
    0x09691a75 0x00 03 GLIBC_2.2.5
    0x069691b4 0x00 02 GLIBC_2.34

Sections:
Idx Name               Size      VMA               LMA               File off  Algn  Flags
  0 .interp            0000001c  0000000000000318  0000000000000318  00000318  2**0  CONTENTS, ALLOC, LOAD, READONLY, DATA
  1 .note.gnu.property 00000030  0000000000000338  0000000000000338  00000338  2**3  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .note.gnu.build-id 00000024  0000000000000368  0000000000000368  00000368  2**2  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .note.ABI-tag      00000020  000000000000038c  000000000000038c  0000038c  2**2  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .gnu.hash          00000024  00000000000003b0  00000000000003b0  000003b0  2**3  CONTENTS, ALLOC, LOAD, READONLY, DATA
  5 .dynsym            000000a8  00000000000003d8  00000000000003d8  000003d8  2**3  CONTENTS, ALLOC, LOAD, READONLY, DATA
  6 .dynstr            0000008d  0000000000000480  0000000000000480  00000480  2**0  CONTENTS, ALLOC, LOAD, READONLY, DATA
  7 .gnu.version       0000000e  000000000000050e  000000000000050e  0000050e  2**1  CONTENTS, ALLOC, LOAD, READONLY, DATA
  8 .gnu.version_r     00000030  0000000000000520  0000000000000520  00000520  2**3  CONTENTS, ALLOC, LOAD, READONLY, DATA
  9 .rela.dyn          000000c0  0000000000000550  0000000000000550  00000550  2**3  CONTENTS, ALLOC, LOAD, READONLY, DATA
 10 .rela.plt          00000018  0000000000000610  0000000000000610  00000610  2**3  CONTENTS, ALLOC, LOAD, READONLY, DATA
 11 .init              0000001b  0000000000001000  0000000000001000  00001000  2**2  CONTENTS, ALLOC, LOAD, READONLY, CODE
 12 .plt               00000020  0000000000001020  0000000000001020  00001020  2**4  CONTENTS, ALLOC, LOAD, READONLY, CODE
 13 .plt.got           00000010  0000000000001040  0000000000001040  00001040  2**4  CONTENTS, ALLOC, LOAD, READONLY, CODE
 14 .plt.sec           00000010  0000000000001050  0000000000001050  00001050  2**4  CONTENTS, ALLOC, LOAD, READONLY, CODE
 15 .text              00000117  0000000000001060  0000000000001060  00001060  2**4  CONTENTS, ALLOC, LOAD, READONLY, CODE
 16 .fini              0000000d  0000000000001178  0000000000001178  00001178  2**2  CONTENTS, ALLOC, LOAD, READONLY, CODE
 17 .rodata            0000000b  0000000000002000  0000000000002000  00002000  2**2  CONTENTS, ALLOC, LOAD, READONLY, DATA
 18 .eh_frame_hdr      0000003c  000000000000200c  000000000000200c  0000200c  2**2  CONTENTS, ALLOC, LOAD, READONLY, DATA
 19 .eh_frame          000000cc  0000000000002048  0000000000002048  00002048  2**3  CONTENTS, ALLOC, LOAD, READONLY, DATA
 20 .init_array        00000008  0000000000003db8  0000000000003db8  00002db8  2**3  CONTENTS, ALLOC, LOAD, DATA
 21 .fini_array        00000008  0000000000003dc0  0000000000003dc0  00002dc0  2**3  CONTENTS, ALLOC, LOAD, DATA
 22 .dynamic           000001f0  0000000000003dc8  0000000000003dc8  00002dc8  2**3  CONTENTS, ALLOC, LOAD, DATA
 23 .got               00000048  0000000000003fb8  0000000000003fb8  00002fb8  2**3  CONTENTS, ALLOC, LOAD, DATA
 24 .data              00000010  0000000000004000  0000000000004000  00003000  2**3  CONTENTS, ALLOC, LOAD, DATA
 25 .bss               00000008  0000000000004010  0000000000004010  00003010  2**0  ALLOC
 26 .comment           0000002d  0000000000000000  0000000000000000  00003010  2**0  CONTENTS, READONLY
 27 .gnu_debuglink     00000010  0000000000000000  0000000000000000  00003040  2**2  CONTENTS, READONLY
SYMBOL TABLE:
0000000000000000 l    df *ABS*  0000000000000000 Scrt1.o
000000000000038c l     O .note.ABI-tag  0000000000000020 __abi_tag
0000000000000000 l    df *ABS*  0000000000000000 crtstuff.c
0000000000001090 l     F .text  0000000000000000 deregister_tm_clones
00000000000010c0 l     F .text  0000000000000000 register_tm_clones
0000000000001100 l     F .text  0000000000000000 __do_global_dtors_aux
0000000000004010 l     O .bss   0000000000000001 completed.0
0000000000003dc0 l     O .fini_array    0000000000000000 __do_global_dtors_aux_fini_array_entry
0000000000001140 l     F .text  0000000000000000 frame_dummy
0000000000003db8 l     O .init_array    0000000000000000 __frame_dummy_init_array_entry
0000000000000000 l    df *ABS*  0000000000000000 test.c
0000000000000000 l    df *ABS*  0000000000000000 crtstuff.c
0000000000002110 l     O .eh_frame      0000000000000000 __FRAME_END__
0000000000000000 l    df *ABS*  0000000000000000
0000000000003dc8 l     O .dynamic       0000000000000000 _DYNAMIC
000000000000200c l       .eh_frame_hdr  0000000000000000 __GNU_EH_FRAME_HDR
0000000000003fb8 l     O .got   0000000000000000 _GLOBAL_OFFSET_TABLE_
0000000000000000       F *UND*  0000000000000000 __libc_start_main@GLIBC_2.34
0000000000000000  w      *UND*  0000000000000000 _ITM_deregisterTMCloneTable
0000000000004000  w      .data  0000000000000000 data_start
0000000000000000       F *UND*  0000000000000000 puts@GLIBC_2.2.5
0000000000004010 g       .data  0000000000000000 _edata
0000000000001178 g     F .fini  0000000000000000 .hidden _fini
0000000000001149 g     F .text  000000000000001a hello
0000000000004000 g       .data  0000000000000000 __data_start
0000000000000000  w      *UND*  0000000000000000 __gmon_start__
0000000000004008 g     O .data  0000000000000000 .hidden __dso_handle
0000000000002000 g     O .rodata        0000000000000004 _IO_stdin_used
0000000000004018 g       .bss   0000000000000000 _end
0000000000001060 g     F .text  0000000000000026 _start
0000000000004010 g       .bss   0000000000000000 __bss_start
0000000000001163 g     F .text  0000000000000014 main
0000000000004010 g     O .data  0000000000000000 .hidden __TMC_END__
0000000000000000  w      *UND*  0000000000000000 _ITM_registerTMCloneTable
0000000000000000  w    F *UND*  0000000000000000 __cxa_finalize@GLIBC_2.2.5
0000000000001000 g     F .init  0000000000000000 .hidden _init
```


## <span class="section-num">6</span> 反汇编可执行 Sections {#反汇编可执行-sections}

objdump -d 反汇编所有可执行executable sections的内容：

```shell
root@lima-ebpf-dev:~# objdump -w -d hello

hello:     file format elf64-x86-64


Disassembly of section .init:

0000000000001000 <_init>:
    1000:       f3 0f 1e fa             endbr64
    1004:       48 83 ec 08             sub    $0x8,%rsp
    1008:       48 8b 05 d9 2f 00 00    mov    0x2fd9(%rip),%rax        # 3fe8 <__gmon_start__@Base>
    100f:       48 85 c0                test   %rax,%rax
    1012:       74 02                   je     1016 <_init+0x16>
    1014:       ff d0                   call   *%rax
    1016:       48 83 c4 08             add    $0x8,%rsp
    101a:       c3                      ret

Disassembly of section .plt:

0000000000001020 <.plt>:
    1020:       ff 35 9a 2f 00 00       push   0x2f9a(%rip)        # 3fc0 <_GLOBAL_OFFSET_TABLE_+0x8>
    1026:       f2 ff 25 9b 2f 00 00    bnd jmp *0x2f9b(%rip)        # 3fc8 <_GLOBAL_OFFSET_TABLE_+0x10>
    102d:       0f 1f 00                nopl   (%rax)
    1030:       f3 0f 1e fa             endbr64
    1034:       68 00 00 00 00          push   $0x0
    1039:       f2 e9 e1 ff ff ff       bnd jmp 1020 <_init+0x20>
    103f:       90                      nop

Disassembly of section .plt.got:

0000000000001040 <__cxa_finalize@plt>:
    1040:       f3 0f 1e fa             endbr64
    1044:       f2 ff 25 ad 2f 00 00    bnd jmp *0x2fad(%rip)        # 3ff8 <__cxa_finalize@GLIBC_2.2.5>
    104b:       0f 1f 44 00 00          nopl   0x0(%rax,%rax,1)

Disassembly of section .plt.sec:

0000000000001050 <puts@plt>:
    1050:       f3 0f 1e fa             endbr64
    1054:       f2 ff 25 75 2f 00 00    bnd jmp *0x2f75(%rip)        # 3fd0 <puts@GLIBC_2.2.5>
    105b:       0f 1f 44 00 00          nopl   0x0(%rax,%rax,1)

Disassembly of section .text:

0000000000001060 <_start>:
    1060:       f3 0f 1e fa             endbr64
    1064:       31 ed                   xor    %ebp,%ebp
    1066:       49 89 d1                mov    %rdx,%r9
    1069:       5e                      pop    %rsi
    106a:       48 89 e2                mov    %rsp,%rdx
    106d:       48 83 e4 f0             and    $0xfffffffffffffff0,%rsp
    1071:       50                      push   %rax
    1072:       54                      push   %rsp
    1073:       45 31 c0                xor    %r8d,%r8d
    1076:       31 c9                   xor    %ecx,%ecx
    1078:       48 8d 3d e4 00 00 00    lea    0xe4(%rip),%rdi        # 1163 <main>
    107f:       ff 15 53 2f 00 00       call   *0x2f53(%rip)        # 3fd8 <__libc_start_main@GLIBC_2.34>
    1085:       f4                      hlt
    1086:       66 2e 0f 1f 84 00 00 00 00 00   cs nopw 0x0(%rax,%rax,1)

0000000000001090 <deregister_tm_clones>:
    1090:       48 8d 3d 79 2f 00 00    lea    0x2f79(%rip),%rdi        # 4010 <__TMC_END__>
    1097:       48 8d 05 72 2f 00 00    lea    0x2f72(%rip),%rax        # 4010 <__TMC_END__>
    109e:       48 39 f8                cmp    %rdi,%rax
    10a1:       74 15                   je     10b8 <deregister_tm_clones+0x28>
    10a3:       48 8b 05 36 2f 00 00    mov    0x2f36(%rip),%rax        # 3fe0 <_ITM_deregisterTMCloneTable@Base>
    10aa:       48 85 c0                test   %rax,%rax
    10ad:       74 09                   je     10b8 <deregister_tm_clones+0x28>
    10af:       ff e0                   jmp    *%rax
    10b1:       0f 1f 80 00 00 00 00    nopl   0x0(%rax)
    10b8:       c3                      ret
    10b9:       0f 1f 80 00 00 00 00    nopl   0x0(%rax)

00000000000010c0 <register_tm_clones>:
    10c0:       48 8d 3d 49 2f 00 00    lea    0x2f49(%rip),%rdi        # 4010 <__TMC_END__>
    10c7:       48 8d 35 42 2f 00 00    lea    0x2f42(%rip),%rsi        # 4010 <__TMC_END__>
    10ce:       48 29 fe                sub    %rdi,%rsi
    10d1:       48 89 f0                mov    %rsi,%rax
    10d4:       48 c1 ee 3f             shr    $0x3f,%rsi
    10d8:       48 c1 f8 03             sar    $0x3,%rax
    10dc:       48 01 c6                add    %rax,%rsi
    10df:       48 d1 fe                sar    %rsi
    10e2:       74 14                   je     10f8 <register_tm_clones+0x38>
    10e4:       48 8b 05 05 2f 00 00    mov    0x2f05(%rip),%rax        # 3ff0 <_ITM_registerTMCloneTable@Base>
    10eb:       48 85 c0                test   %rax,%rax
    10ee:       74 08                   je     10f8 <register_tm_clones+0x38>
    10f0:       ff e0                   jmp    *%rax
    10f2:       66 0f 1f 44 00 00       nopw   0x0(%rax,%rax,1)
    10f8:       c3                      ret
    10f9:       0f 1f 80 00 00 00 00    nopl   0x0(%rax)

0000000000001100 <__do_global_dtors_aux>:
    1100:       f3 0f 1e fa             endbr64
    1104:       80 3d 05 2f 00 00 00    cmpb   $0x0,0x2f05(%rip)        # 4010 <__TMC_END__>
    110b:       75 2b                   jne    1138 <__do_global_dtors_aux+0x38>
    110d:       55                      push   %rbp
    110e:       48 83 3d e2 2e 00 00 00         cmpq   $0x0,0x2ee2(%rip)        # 3ff8 <__cxa_finalize@GLIBC_2.2.5>
    1116:       48 89 e5                mov    %rsp,%rbp
    1119:       74 0c                   je     1127 <__do_global_dtors_aux+0x27>
    111b:       48 8b 3d e6 2e 00 00    mov    0x2ee6(%rip),%rdi        # 4008 <__dso_handle>
    1122:       e8 19 ff ff ff          call   1040 <__cxa_finalize@plt>
    1127:       e8 64 ff ff ff          call   1090 <deregister_tm_clones>
    112c:       c6 05 dd 2e 00 00 01    movb   $0x1,0x2edd(%rip)        # 4010 <__TMC_END__>
    1133:       5d                      pop    %rbp
    1134:       c3                      ret
    1135:       0f 1f 00                nopl   (%rax)
    1138:       c3                      ret
    1139:       0f 1f 80 00 00 00 00    nopl   0x0(%rax)

0000000000001140 <frame_dummy>:
    1140:       f3 0f 1e fa             endbr64
    1144:       e9 77 ff ff ff          jmp    10c0 <register_tm_clones>

0000000000001149 <hello>:
    1149:       f3 0f 1e fa             endbr64
    114d:       55                      push   %rbp
    114e:       48 89 e5                mov    %rsp,%rbp
    1151:       48 8d 05 ac 0e 00 00    lea    0xeac(%rip),%rax        # 2004 <_IO_stdin_used+0x4>
    1158:       48 89 c7                mov    %rax,%rdi
    115b:       e8 f0 fe ff ff          call   1050 <puts@plt>
    1160:       90                      nop
    1161:       5d                      pop    %rbp
    1162:       c3                      ret

0000000000001163 <main>:
    1163:       f3 0f 1e fa             endbr64
    1167:       55                      push   %rbp
    1168:       48 89 e5                mov    %rsp,%rbp
    116b:       e8 d9 ff ff ff          call   1149 <hello>
    1170:       b8 00 00 00 00          mov    $0x0,%eax
    1175:       5d                      pop    %rbp
    1176:       c3                      ret

Disassembly of section .fini:

0000000000001178 <_fini>:
    1178:       f3 0f 1e fa             endbr64
    117c:       48 83 ec 08             sub    $0x8,%rsp
    1180:       48 83 c4 08             add    $0x8,%rsp
    1184:       c3                      ret
root@lima-ebpf-dev:~#
```

objdump --disassemble=&lt;sym&gt; 反汇编指定 symbol 的内容：

```shell
root@lima-ebpf-dev:~# objdump -w --disassemble=hello hello

hello:     file format elf64-x86-64

Disassembly of section .init:

Disassembly of section .plt:

Disassembly of section .plt.got:

Disassembly of section .plt.sec:

Disassembly of section .text:

0000000000001149 <hello>:
    1149:       f3 0f 1e fa             endbr64
    114d:       55                      push   %rbp
    114e:       48 89 e5                mov    %rsp,%rbp
    1151:       48 8d 05 ac 0e 00 00    lea    0xeac(%rip),%rax        # 2004 <_IO_stdin_used+0x4>
    1158:       48 89 c7                mov    %rax,%rdi
    115b:       e8 f0 fe ff ff          call   1050 <puts@plt>
    1160:       90                      nop
    1161:       5d                      pop    %rbp
    1162:       c3                      ret

Disassembly of section .fini:
root@lima-ebpf-dev:~#
```

objdump -S/--source 反汇编时显示源文件的内容：

-   -S 使用的是 build-id 而非gnu debug link机制，必须将debug file放到`/usr/lib/debug/.build-id`目录下时，才会显示源文件内容。

<!--listend-->

```shell
root@lima-ebpf-dev:~# strace -e openat objdump -S hello  |& grep /usr/lib/debug
openat(AT_FDCWD, "/usr/lib/debug/hello.debug", O_RDONLY) = 4
openat(AT_FDCWD, "/usr/lib/debug/hello.debug", O_RDONLY) = 5
openat(AT_FDCWD, "/usr/lib/debug/hello.debug", O_RDONLY) = 5
openat(AT_FDCWD, "/usr/lib/debug/.build-id/7e/31292c839740f24092f371f1e85cd9ad74a79b.debug", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/debug/usr/.build-id/7e/31292c839740f24092f371f1e85cd9ad74a79b.debug", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/debug/.build-id/7e/31292c839740f24092f371f1e85cd9ad74a79b.debug", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/debug/usr/.build-id/7e/31292c839740f24092f371f1e85cd9ad74a79b.debug", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/debug/.build-id/7e/31292c839740f24092f371f1e85cd9ad74a79b.debug", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/debug/usr/.build-id/7e/31292c839740f24092f371f1e85cd9ad74a79b.debug", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/debug/root/hello.debug", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/debug/usr/root/hello.debug", O_RDONLY) = -1 ENOENT (No such file or directory)
root@lima-ebpf-dev:~#

root@lima-ebpf-dev:~# objdump -S --disassemble=hello hello

hello:     file format elf64-x86-64


Disassembly of section .init:

Disassembly of section .plt:

Disassembly of section .plt.got:

Disassembly of section .plt.sec:

Disassembly of section .text:

0000000000001149 <hello>:
    1149:       f3 0f 1e fa             endbr64
    114d:       55                      push   %rbp
    114e:       48 89 e5                mov    %rsp,%rbp
    1151:       48 8d 05 ac 0e 00 00    lea    0xeac(%rip),%rax        # 2004 <_IO_stdin_used+0x4>
    1158:       48 89 c7                mov    %rax,%rdi
    115b:       e8 f0 fe ff ff          call   1050 <puts@plt>
    1160:       90                      nop
    1161:       5d                      pop    %rbp
    1162:       c3                      ret

Disassembly of section .fini:

root@lima-ebpf-dev:~# mv /usr/lib/debug/hello.debug  /usr/lib/debug/.build-id/7e/31292c839740f24092f371f1e85cd9ad74a79b.debug
root@lima-ebpf-dev:~# objdump -S --disassemble=hello hello

hello:     file format elf64-x86-64


Disassembly of section .init:

Disassembly of section .plt:

Disassembly of section .plt.got:

Disassembly of section .plt.sec:

Disassembly of section .text:

0000000000001149 <hello>:
#include <stdio.h>

void hello(void){
    1149:       f3 0f 1e fa             endbr64
    114d:       55                      push   %rbp
    114e:       48 89 e5                mov    %rsp,%rbp
        printf("hello!\n");
    1151:       48 8d 05 ac 0e 00 00    lea    0xeac(%rip),%rax        # 2004 <_IO_stdin_used+0x4>
    1158:       48 89 c7                mov    %rax,%rdi
    115b:       e8 f0 fe ff ff          call   1050 <puts@plt>
}
    1160:       90                      nop
    1161:       5d                      pop    %rbp
    1162:       c3                      ret

Disassembly of section .fini:
root@lima-ebpf-dev:~#
```


## <span class="section-num">7</span> 显示 Sections 内容 {#显示-sections-内容}

objdump -s 十六进制显示对象文件各 Section 的内容：

```shell
root@lima-ebpf-dev:~# objdump -s hello |tail
 3fe8 00000000 00000000 00000000 00000000  ................
 3ff8 00000000 00000000                    ........
Contents of section .data:
 4000 00000000 00000000 08400000 00000000  .........@......
Contents of section .comment:
 0000 4743433a 20285562 756e7475 2031312e  GCC: (Ubuntu 11.
 0010 332e302d 31756275 6e747531 7e32322e  3.0-1ubuntu1~22.
 0020 30342e31 29203131 2e332e30 00        04.1) 11.3.0.
Contents of section .gnu_debuglink:
 0000 68656c6c 6f2e6465 62756700 a3c55b6a  hello.debug...[j
```

通过添加-j SECTION来只显示指定 SECTION 的内容：

```shell
root@lima-ebpf-dev:~# objdump -s hello -j .comment

hello:     file format elf64-x86-64

Contents of section .comment:
 0000 4743433a 20285562 756e7475 2031312e  GCC: (Ubuntu 11.
 0010 342e302d 31756275 6e747531 7e32322e  4.0-1ubuntu1~22.
 0020 30342920 31312e34 2e3000             04) 11.4.0.
```


## <span class="section-num">8</span> 显示符号表 {#显示符号表}

objdump -t/--syms 显示符号表：

-   会在/usr/lib/debug目录下查找hello.debug或者使用.build-id来查找 debug 文件，然后提取符号表。所以，即使二进制本身被 strip ，只要有 debug文件，也能显示符号表。

<!--listend-->

```shell
root@lima-ebpf-dev:~# objdump -t hello

hello:     file format elf64-x86-64

SYMBOL TABLE:
0000000000000000 l    df *ABS*  0000000000000000 Scrt1.o
000000000000038c l     O .note.ABI-tag  0000000000000020 __abi_tag
0000000000000000 l    df *ABS*  0000000000000000 crtstuff.c
0000000000001090 l     F .text  0000000000000000 deregister_tm_clones
00000000000010c0 l     F .text  0000000000000000 register_tm_clones
0000000000001100 l     F .text  0000000000000000 __do_global_dtors_aux
0000000000004010 l     O .bss   0000000000000001 completed.0
0000000000003dc0 l     O .fini_array    0000000000000000 __do_global_dtors_aux_fini_array_entry
0000000000001140 l     F .text  0000000000000000 frame_dummy
0000000000003db8 l     O .init_array    0000000000000000 __frame_dummy_init_array_entry
0000000000000000 l    df *ABS*  0000000000000000 test.c
0000000000000000 l    df *ABS*  0000000000000000 crtstuff.c
0000000000002110 l     O .eh_frame      0000000000000000 __FRAME_END__
0000000000000000 l    df *ABS*  0000000000000000
0000000000003dc8 l     O .dynamic       0000000000000000 _DYNAMIC
000000000000200c l       .eh_frame_hdr  0000000000000000 __GNU_EH_FRAME_HDR
0000000000003fb8 l     O .got   0000000000000000 _GLOBAL_OFFSET_TABLE_
0000000000000000       F *UND*  0000000000000000 __libc_start_main@GLIBC_2.34
0000000000000000  w      *UND*  0000000000000000 _ITM_deregisterTMCloneTable
0000000000004000  w      .data  0000000000000000 data_start
0000000000000000       F *UND*  0000000000000000 puts@GLIBC_2.2.5
0000000000004010 g       .data  0000000000000000 _edata
0000000000001178 g     F .fini  0000000000000000 .hidden _fini
0000000000001149 g     F .text  000000000000001a hello
0000000000004000 g       .data  0000000000000000 __data_start
0000000000000000  w      *UND*  0000000000000000 __gmon_start__
0000000000004008 g     O .data  0000000000000000 .hidden __dso_handle
0000000000002000 g     O .rodata        0000000000000004 _IO_stdin_used
0000000000004018 g       .bss   0000000000000000 _end
0000000000001060 g     F .text  0000000000000026 _start
0000000000004010 g       .bss   0000000000000000 __bss_start
0000000000001163 g     F .text  0000000000000014 main
0000000000004010 g     O .data  0000000000000000 .hidden __TMC_END__
0000000000000000  w      *UND*  0000000000000000 _ITM_registerTMCloneTable
0000000000000000  w    F *UND*  0000000000000000 __cxa_finalize@GLIBC_2.2.5
0000000000001000 g     F .init  0000000000000000 .hidden _init

root@lima-ebpf-dev:~#

root@lima-ebpf-dev:~# strace -f  -e open,openat objdump -t hello  |& grep /lib/debug/
openat(AT_FDCWD, "/usr/lib/debug/hello.debug", O_RDONLY) = 4
openat(AT_FDCWD, "/usr/lib/debug/hello.debug", O_RDONLY) = 5
openat(AT_FDCWD, "/usr/lib/debug/hello.debug", O_RDONLY) = 5
openat(AT_FDCWD, "/usr/lib/debug/.build-id/7e/31292c839740f24092f371f1e85cd9ad74a79b.debug", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/debug/usr/.build-id/7e/31292c839740f24092f371f1e85cd9ad74a79b.debug", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/debug/.build-id/7e/31292c839740f24092f371f1e85cd9ad74a79b.debug", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/debug/usr/.build-id/7e/31292c839740f24092f371f1e85cd9ad74a79b.debug", O_RDONLY) = -1 ENOENT (No such file or directory)
```

objdump -T/--dynamic-syms： 显示动态符号表：

```shell
root@lima-ebpf-dev:~# objdump -T hello

hello:     file format elf64-x86-64

DYNAMIC SYMBOL TABLE:
0000000000000000      DF *UND*  0000000000000000 (GLIBC_2.34) __libc_start_main
0000000000000000  w   D  *UND*  0000000000000000  Base        _ITM_deregisterTMCloneTable
0000000000000000      DF *UND*  0000000000000000 (GLIBC_2.2.5) puts
0000000000000000  w   D  *UND*  0000000000000000  Base        __gmon_start__
0000000000000000  w   D  *UND*  0000000000000000  Base        _ITM_registerTMCloneTable
0000000000000000  w   DF *UND*  0000000000000000 (GLIBC_2.2.5) __cxa_finalize


root@lima-ebpf-dev:~#
```


## <span class="section-num">9</span> 显示调试符号表 DWARF {#显示调试符号表-dwarf}

objdump -g/-W 显示文件的 DWARF 格式的 debug 信息：

-   直接从文件中读，或者根据debug-link/build-id从系统`/usr/lib/debug`读取 debug 文件。
    ```shell
    root@lima-ebpf-dev:~# objdump -w -g  hello
    Contents of the .debug_aranges section (loaded from /usr/lib/debug/hello.debug):

      Length:                   44
      Version:                  2
      Offset into .debug_info:  0x0
      Pointer Size:             8
      Segment Size:             0

        Address            Length
        0000000000001149 000000000000002e
        0000000000000000 0000000000000000

    Contents of the .debug_info section (loaded from /usr/lib/debug/hello.debug):

      Compilation Unit @ offset 0x0:
       Length:        0xa2 (32-bit)
       Version:       5
       Unit Type:     DW_UT_compile (1)
       Abbrev Offset: 0x0
       Pointer Size:  8
     <0><c>: Abbrev Number: 2 (DW_TAG_compile_unit)
        <d>   DW_AT_producer    : (strp) (offset: 0x35): GNU C17 11.3.0 -mtune=generic -march=x86-64 -g -fasynchronous-unwind-tables -fstack-protector-strong -fstack-clash-protection -fcf-protection
        <11>   DW_AT_language    : (data1) 29       (C11)
        <12>   DW_AT_name        : (line_strp) (offset: 0x0): test.c
        <16>   DW_AT_comp_dir    : (line_strp) (offset: 0x7): /root
        <1a>   DW_AT_low_pc      : (addr) 0x1149
        <22>   DW_AT_high_pc     : (data8)
    ...
    Contents of the .gnu_debuglink section (loaded from hello):

      Separate debug info file: hello.debug
      CRC value: 0x6a5bc5a3

    root@lima-ebpf-dev:~#
    ```

objdump --dwarf 可以指定要打印的 DWARF 内容类型，支持如下参数：-W, --dwarf[a/=abbrev, A/=addr, r/=aranges,
 c/=cu_index, L/=decodedline, f/=frames, F/=frames-interp, g/=gdb_index, i/=info, o/=loc, m/=macro,
 p/=pubnames, t/=pubtypes, R/=Ranges, l/=rawline, s/=str, O/=str-offsets, u/=trace_abbrev, T/=trace_aranges,
 U/=trace_info]

例如打印内存地址和源码行之间的关系；

```shell
root@lima-ebpf-dev:~# objdump --dwarf=decodedline hello
Contents of the .debug_line section (loaded from /usr/lib/debug/.build-id/7e/31292c839740f24092f371f1e85cd9ad74a79b.debug):

test.c:
File name                            Line number    Starting address    View    Stmt
test.c                                         3              0x1149               x
test.c                                         4              0x1151               x
test.c                                         5              0x1160               x
test.c                                         6              0x1163               x
test.c                                         7              0x116b               x
test.c                                         8              0x1170               x
test.c                                         9              0x1175               x
test.c                                         -              0x1177



hello:     file format elf64-x86-64

root@lima-ebpf-dev:~#
```
