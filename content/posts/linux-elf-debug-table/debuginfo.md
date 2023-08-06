---
title: "Linux elf 调试符号表（.debug_XX)"
author: ["opsnull"]
date: 2023-08-06T00:00:00+08:00
lastmod: 2023-08-06T21:12:32+08:00
tags: ["linux", "elf", "debug"]
categories: ["debug"]
draft: false
series: ["elf-debug"]
series_order: 2
---

介绍Linux elf二进制文件的调试符号表（.debug_XX）生成和管理机制。

<!--more-->

Linux 使用 ELF 格式类保存可执行二进制文件、共享库文件和 debuginfo文件。

gdb 依赖调试符号表来进行 stack unwinding/单步调试/内存地址和源文件对应关系。

使用`bpftrace -lv 'uprobe:/bin/bash:readline'`来显示 readline 函数参数时，也是从调试符号表中解析函数名称和参数信息。

-   如果 bpftrace 查不到调试符号表，则会报错： `No DWARF found for XX，cannot show parameter info`


## <span class="section-num">1</span> 生成调试符号表 {#生成调试符号表}

由于 debuginfo 通常比较大，而且一般只在 debug 时使用。所以编译器默认不生成它们。在编译二进制时，通过指定 -g 选项，可以在生成的二进制中包含.debug_XX开头的调试符号表：

-   例如： .debug_info, .debug_abbrev, .debug_line, .debug_frame, .debug_str,
    .debug_loc 等，它们符合 DWARF 格式标准；
-   可以使用`readelf -w file或 objdump -g` 命令来查看.
-   使用 -g3 可以包含 macro definitions（默认 -g2）；

<!--listend-->

```shell
# gcc -g test.c -o hello # -g 指定生成调试符号表

# objdump -h hello

hello:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .interp       0000001c  0000000000400238  0000000000400238  00000238  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  1 .note.ABI-tag 00000020  0000000000400254  0000000000400254  00000254  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .note.gnu.build-id 00000024  0000000000400274  0000000000400274  00000274  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .gnu.hash     0000001c  0000000000400298  0000000000400298  00000298  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .dynsym       00000078  00000000004002b8  00000000004002b8  000002b8  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  5 .dynstr       00000043  0000000000400330  0000000000400330  00000330  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  6 .gnu.version  0000000a  0000000000400374  0000000000400374  00000374  2**1
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  7 .gnu.version_r 00000020  0000000000400380  0000000000400380  00000380  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  8 .rela.dyn     00000018  00000000004003a0  00000000004003a0  000003a0  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  9 .rela.plt     00000060  00000000004003b8  00000000004003b8  000003b8  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 10 .init         0000001a  0000000000400418  0000000000400418  00000418  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 11 .plt          00000050  0000000000400440  0000000000400440  00000440  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 12 .text         00000192  0000000000400490  0000000000400490  00000490  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 13 .fini         00000009  0000000000400624  0000000000400624  00000624  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 14 .rodata       0000001d  0000000000400630  0000000000400630  00000630  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 15 .eh_frame_hdr 0000003c  0000000000400650  0000000000400650  00000650  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 16 .eh_frame     00000114  0000000000400690  0000000000400690  00000690  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 17 .init_array   00000008  0000000000600e10  0000000000600e10  00000e10  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 18 .fini_array   00000008  0000000000600e18  0000000000600e18  00000e18  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 19 .jcr          00000008  0000000000600e20  0000000000600e20  00000e20  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 20 .dynamic      000001d0  0000000000600e28  0000000000600e28  00000e28  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 21 .got          00000008  0000000000600ff8  0000000000600ff8  00000ff8  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 22 .got.plt      00000038  0000000000601000  0000000000601000  00001000  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 23 .data         00000004  0000000000601038  0000000000601038  00001038  2**0
                  CONTENTS, ALLOC, LOAD, DATA
 24 .bss          00000004  000000000060103c  000000000060103c  0000103c  2**0
                  ALLOC
 25 .comment      0000002d  0000000000000000  0000000000000000  0000103c  2**0
                  CONTENTS, READONLY
 26 .debug_aranges 00000030  0000000000000000  0000000000000000  00001069  2**0  # 以下为各种调试符号表
                  CONTENTS, READONLY, DEBUGGING
 27 .debug_info   000000aa  0000000000000000  0000000000000000  00001099  2**0
                  CONTENTS, READONLY, DEBUGGING
 28 .debug_abbrev 00000058  0000000000000000  0000000000000000  00001143  2**0
                  CONTENTS, READONLY, DEBUGGING
 29 .debug_line   0000003e  0000000000000000  0000000000000000  0000119b  2**0
                  CONTENTS, READONLY, DEBUGGING
 30 .debug_str    000000ba  0000000000000000  0000000000000000  000011d9  2**0
                  CONTENTS, READONLY, DEBUGGING
```


## <span class="section-num">2</span> 创建和保存 debuginfo 文件 {#创建和保存-debuginfo-文件}

可以使用`objcopy --only-keep-debug hello hello.debug`来创建单独的 debug 文件 hello.debug，该文件 `同时包含`
`.symtab` 和各种 `.debug_XX` Sections：

```shell
# du -sh hello
12K     hello

# objcopy --only-keep-debug hello hello.debug

# du -sh hello.debug
8.0K    hello.debug
```

根据 hello 中 `.note.gnu.build-id` 值放到系统标准 debug 文件目录`/usr/lib/debug/.build-id/xx/XXX.debug`中，这样后续`gdb/objdump/readelf`等都会自动读取。

-   使用`readelf -n XX`命令查看 build-id 的内容。

<!--listend-->

```shell
root@lima-ebpf-dev:~# readelf -n hello

Displaying notes found in: .note.gnu.property
  Owner                Data size        Description
  GNU                  0x00000020       NT_GNU_PROPERTY_TYPE_0
      Properties: x86 feature: IBT, SHSTK
        x86 ISA needed: x86-64-baseline

Displaying notes found in: .note.gnu.build-id
  Owner                Data size        Description
  GNU                  0x00000014       NT_GNU_BUILD_ID (unique build ID bitstring)
    Build ID: 7e31292c839740f24092f371f1e85cd9ad74a79b

Displaying notes found in: .note.ABI-tag
  Owner                Data size        Description
  GNU                  0x00000010       NT_GNU_ABI_TAG (ABI version tag)
    OS: Linux, ABI: 3.2.0

# 根据 build-id 前两位，创建一个目录；
root@lima-ebpf-dev:~# mkdir /usr/lib/debug/.build-id/7e

# 文件名为 build-id 去掉前两位后的内容 + .debug
root@lima-ebpf-dev:~# mv hello.debug  /usr/lib/debug/.build-id/7e/31292c839740f24092f371f1e85cd9ad74a79b.debug
```

也可以在生成hello.debug文件后，使用命令`objcopy --add-gnu-debuglink=hello.debug hello`在 hello 中添加一个
`'.gnu_debuglink'` Sections ，然后将hello.debug文件放置到`/usr/lib/debug/hello.debug`位置，这样后续
`gdb/objdump/readelf` 也会自动查找使用。

-   debuglink 包含 32 位 CRC 校验码, GDB 可以用来确认找到的 debug 文件是匹配的.
-   使用`readelf --string-dump=.gnu_debuglink hello`命令来查看 debuglink 的内容。

<!--listend-->

```shell
# objcopy --add-gnu-debuglink=hello.debug hello
# objdump -h hello |grep .gnu_debug
 31 .gnu_debuglink 00000010  0000000000000000  0000000000000000  00001293  2**0

#readelf --string-dump=.gnu_debuglink hello

String dump of section '.gnu_debuglink':
  [     0]  hello.debug
```

由于编译时自动生成 build-id 且不会重复，所以建议使用 build-id 而非 debuglink 机制。

-   objdump -S 只认 build-id。

最后，使用`strip -g BINARY`命令将 BINARY 中的调试符号表删除，从而减轻二进制文件体积：

-   `strip -g` ：删除各种.debug_XX开头的调试符号表，这些表中包含源文件、行号等与内存地址的关系，用于支持单步调试。
-   `strip -s/--strip-all` ：同时删除.symtab和各种.debug_XX调试符号表；

<!--listend-->

```shell
#strip -g hello

#du -sh hello
12K     hello

#objdump -h hello

hello:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .interp       0000001c  0000000000400238  0000000000400238  00000238  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  1 .note.ABI-tag 00000020  0000000000400254  0000000000400254  00000254  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .note.gnu.build-id 00000024  0000000000400274  0000000000400274  00000274  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .gnu.hash     0000001c  0000000000400298  0000000000400298  00000298  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .dynsym       00000078  00000000004002b8  00000000004002b8  000002b8  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  5 .dynstr       00000043  0000000000400330  0000000000400330  00000330  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  6 .gnu.version  0000000a  0000000000400374  0000000000400374  00000374  2**1
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  7 .gnu.version_r 00000020  0000000000400380  0000000000400380  00000380  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  8 .rela.dyn     00000018  00000000004003a0  00000000004003a0  000003a0  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  9 .rela.plt     00000060  00000000004003b8  00000000004003b8  000003b8  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 10 .init         0000001a  0000000000400418  0000000000400418  00000418  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 11 .plt          00000050  0000000000400440  0000000000400440  00000440  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 12 .text         00000192  0000000000400490  0000000000400490  00000490  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 13 .fini         00000009  0000000000400624  0000000000400624  00000624  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 14 .rodata       0000001d  0000000000400630  0000000000400630  00000630  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 15 .eh_frame_hdr 0000003c  0000000000400650  0000000000400650  00000650  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 16 .eh_frame     00000114  0000000000400690  0000000000400690  00000690  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 17 .init_array   00000008  0000000000600e10  0000000000600e10  00000e10  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 18 .fini_array   00000008  0000000000600e18  0000000000600e18  00000e18  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 19 .jcr          00000008  0000000000600e20  0000000000600e20  00000e20  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 20 .dynamic      000001d0  0000000000600e28  0000000000600e28  00000e28  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 21 .got          00000008  0000000000600ff8  0000000000600ff8  00000ff8  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 22 .got.plt      00000038  0000000000601000  0000000000601000  00001000  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 23 .data         00000004  0000000000601038  0000000000601038  00001038  2**0
                  CONTENTS, ALLOC, LOAD, DATA
 24 .bss          00000004  000000000060103c  000000000060103c  0000103c  2**0
                  ALLOC
 25 .comment      0000002d  0000000000000000  0000000000000000  0000103c  2**0
                  CONTENTS, READONLY
 26 .gnu_debuglink 00000010  0000000000000000  0000000000000000  00001069  2**0
                  CONTENTS, READONLY
```

和 strip 相反的工具是`elfutils`包提供的`eu-unstrip`工具，它可以将exec binary和 debug file 合并到一起，形成一个包含 debug 信息的exec binary文件：

```shell
# 格式
eu-unstrip -f <executablefilename> <symbolefilename.debug> -o <newoutputfilename>

# 示例
root@lima-ebpf-dev:/sys/kernel/debug/tracing# readelf -n /usr/bin/bash |grep Build
    Build ID: 33a5554034feb2af38e8c75872058883b2988bc5
root@lima-ebpf-dev:/sys/kernel/debug/tracing# eu-unstrip -f /usr/bin/bash /usr/lib/debug/.build-id/33/a5554034feb2af38e8c75872058883b2988bc5.debug -o /tmp/bash
```


## <span class="section-num">3</span> 查找 debuginfo 文件 {#查找-debuginfo-文件}

`readelf/objdump/gdb/bpftrace` 等工具从系统`/usr/lib/debug/`查找二进制对应的 debuginfo 文件，一般使用两种机制：

-   `.note.gnu.build-id` Section，使用命令 `readelf -n hello`查看该 Section 内容；
-   `.gnu_debuglink` Section，使用命令`readelf --string-dump=.gnu_debuglink hello`来查看内容，debug link 包含 32
    位 CRC 校验码, gdb 可以用来确认找到的 debug 文件是匹配的；

例如 gdb 要 debug 的`/usr/bin/ls`的 debuglink 是ls.debug,而且 build-id 是 hex abcdef1234，全局 debug 目录是
/usr/include/debug, 则 gdb 查找debug info文件的顺序如下：

-   `/usr/lib/debug/.build-id/ab/cdef1234.debug`
    -   注意：.build-id 下的目录是build ID hex值的`前两位`，目录下的文件名是去掉前两位后的内容+.debug;
-   `/usr/bin/ls.debug`
-   `/usr/bin/.debug/ls.debug`
-   `/usr/lib/debug/usr/bin/ls.debug`

gdb 可以通过参数 --with-separate-debug-dir 设置查找 debug info dir：

```shell
(gdb) show debug-file-directory
The directory where separate debug symbols are searched for is "/usr/lib/debug".
(gdb)
```

readelf 查找hello.debug文件的路径：

-   它不会查找/usr/lib/debug目录下的各种 bin 目录。

<!--listend-->

```shell
root@lima-ebpf-dev:~# readelf -n hello

Displaying notes found in: .note.gnu.property
  Owner                Data size        Description
  GNU                  0x00000020       NT_GNU_PROPERTY_TYPE_0
      Properties: x86 feature: IBT, SHSTK
        x86 ISA needed: x86-64-baseline

Displaying notes found in: .note.gnu.build-id
  Owner                Data size        Description
  GNU                  0x00000014       NT_GNU_BUILD_ID (unique build ID bitstring)
    Build ID: 7e31292c839740f24092f371f1e85cd9ad74a79b

Displaying notes found in: .note.ABI-tag
  Owner                Data size        Description
  GNU                  0x00000010       NT_GNU_ABI_TAG (ABI version tag)
    OS: Linux, ABI: 3.2.0
root@lima-ebpf-dev:~#

root@lima-ebpf-dev:~# strace -e openat readelf -P -wk hello |&grep /usr/lib/debug
readelf: Warning: tried: /usr/lib/debug/usr/hello.debug
readelf: Warning: tried: /usr/lib/debug//root//hello.debug
readelf: Warning: tried: /usr/lib/debug/hello.debug
openat(AT_FDCWD, "/usr/lib/debug/.build-id/7e/31292c839740f24092f371f1e85cd9ad74a79b.debug", O_RDONLY) = 4
```

bpftrace 使用 bcc 的 libclang 来[编译和查找二进制的debug file](https://github.com/iovisor/bcc/blob/master/src/cc/bcc_elf.c#L652C25-L652C25)，也会查找系统缺省 debug目录 `/usr/lib/debug`

-   支持 find_debug_via_debuglink/find_debug_via_buildid。
-   也可以通过BCC_DEBUGINFO_ROOT设置 debug 目录。
-   bpftrace 查找 debuginfo 示例：

<!--listend-->

```shell
root@lima-ebpf-dev:~# strace -e openat bpftrace -l 'uprobe:/bin/bash:readline' -v  |& grep debug
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libdebuginfod.so.1", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/sys/kernel/debug/tracing/available_filter_functions", O_RDONLY) = 3
openat(AT_FDCWD, "/sys/kernel/debug/kprobes/blacklist", O_RDONLY) = 4
openat(AT_FDCWD, "/usr/lib/debug/.build-id/33/a5554034feb2af38e8c75872058883b2988bc5.debug", O_RDONLY) = 4
openat(AT_FDCWD, "/usr/lib/debug/.build-id/33/a5554034feb2af38e8c75872058883b2988bc5.debug", O_RDONLY) = 3
root@lima-ebpf-dev:~#
```

如果未找到，但是 debuginfod 被开启，则gdb/bpftrace尝试从 debuginfod 服务器下载 debuginfo 文件。

以CentsOS7 /bin/bash为例：

-   二进制的被 strip 过，所以只有.dynsym符号表，而没有.symtab 符号表。
    -   .dynsym ( dynamic symbols, which can be used by another program）；
    -   .symtab ( “ local ” symbols used by the binary itself only )；
-   二进制缺少.symtab表，且也没有.debug表时，不能使用 uprobe 来插桩函数。
    -   uprobe 插桩只依赖.symtab表，.debug表不是必须的，前者是记录程序中符号（如变量名、函数名）和二进制地址间的关系，而后者是记录源码行号和二进制地址间的关系（用于单步调试）。

<!--listend-->

```shell
# file /usr/bin/bash
/usr/bin/bash: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, BuildID[sha1]=03eac9b3562001644b8685d22f867151b359ecb9, stripped

# readelf -s /usr/bin/bash |grep Symb
Symbol table '.dynsym' contains 2170 entries:

#objdump -h /usr/bin/bash

/usr/bin/bash:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .interp       0000001c  0000000000400238  0000000000400238  00000238  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  1 .note.ABI-tag 00000020  0000000000400254  0000000000400254  00000254  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .note.gnu.build-id 00000024  0000000000400274  0000000000400274  00000274  2**2  # 记录了 build id，后续也用于查找 debug 文件
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .gnu.hash     000036d4  0000000000400298  0000000000400298  00000298  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .dynsym       0000cb70  0000000000403970  0000000000403970  00003970  2**3  # .dynsym 符号表
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  5 .dynstr       00008423  00000000004104e0  00000000004104e0  000104e0  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  6 .gnu.version  000010f4  0000000000418904  0000000000418904  00018904  2**1
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  7 .gnu.version_r 000000b0  00000000004199f8  00000000004199f8  000199f8  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  8 .rela.dyn     000000c0  0000000000419aa8  0000000000419aa8  00019aa8  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  9 .rela.plt     00001398  0000000000419b68  0000000000419b68  00019b68  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 10 .init         0000001a  000000000041af00  000000000041af00  0001af00  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 11 .plt          00000d20  000000000041af20  000000000041af20  0001af20  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 12 .text         0008b5d2  000000000041bc40  000000000041bc40  0001bc40  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 13 .fini         00000009  00000000004a7214  00000000004a7214  000a7214  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
 14 .rodata       0001cd0e  00000000004a7220  00000000004a7220  000a7220  2**5
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 15 .eh_frame_hdr 00003c74  00000000004c3f30  00000000004c3f30  000c3f30  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 16 .eh_frame     0001597c  00000000004c7ba8  00000000004c7ba8  000c7ba8  2**3
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 17 .init_array   00000008  00000000006dddf0  00000000006dddf0  000dddf0  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 18 .fini_array   00000008  00000000006dddf8  00000000006dddf8  000dddf8  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 19 .jcr          00000008  00000000006dde00  00000000006dde00  000dde00  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 20 .dynamic      000001f0  00000000006dde08  00000000006dde08  000dde08  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 21 .got          00000008  00000000006ddff8  00000000006ddff8  000ddff8  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 22 .got.plt      000006a0  00000000006de000  00000000006de000  000de000  2**3
                  CONTENTS, ALLOC, LOAD, DATA
 23 .data         000083f0  00000000006de6a0  00000000006de6a0  000de6a0  2**5
                  CONTENTS, ALLOC, LOAD, DATA
 24 .bss          00005988  00000000006e6aa0  00000000006e6aa0  000e6a90  2**5
                  ALLOC
 25 .gnu_debuglink 00000010  0000000000000000  0000000000000000  000e6a90  2**2  # 记录了 debug 文件的名称和 CRC 校验值
                  CONTENTS, READONLY
 26 .gnu_debugdata 000044c8  0000000000000000  0000000000000000  000e6aa0  2**0
                  CONTENTS, READONLY
```

安装 bash 的 debuginfo 包，提供 debug 文件：

-   安装到系统缺省`/usr/lib/debug`目录下；
-   `/usr/lib/debug/.build-id` 下根据 build-id 来匹配；
-   `/usr/lib/debug/usr/bin/bash.debug` 对应.gnu_debuglink中的文件名；

<!--listend-->

```shell
# rpm -ql bash-debuginfo |head
/usr/lib/debug
/usr/lib/debug/.build-id
/usr/lib/debug/.build-id/03
/usr/lib/debug/.build-id/03/eac9b3562001644b8685d22f867151b359ecb9
/usr/lib/debug/.build-id/03/eac9b3562001644b8685d22f867151b359ecb9.debug
/usr/lib/debug/usr
/usr/lib/debug/usr/bin
/usr/lib/debug/usr/bin/bash.debug
/usr/lib/debug/usr/bin/sh.debug
/usr/src/debug/bash-4.2
```

bash.debug 文件的 build-id 和 bash 文件一致：

-   bash.debug 文件中包含各种以.debug_XX开头的调试符号表。

<!--listend-->

```shell
# file /usr/lib/debug/usr/bin/bash.debug
/usr/lib/debug/usr/bin/bash.debug: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, BuildID[sha1]=03eac9b3562001644b8685d22f867151b359ecb9, not stripped

# objdump -h /usr/lib/debug/bin/bash.debug

/usr/lib/debug/bin/bash.debug:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .interp       0000001c  0000000000400238  0000000000400238  00000238  2**0
                  ALLOC, READONLY
  1 .note.ABI-tag 00000020  0000000000400254  0000000000400238  00000238  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .note.gnu.build-id 00000024  0000000000400274  0000000000400258  00000258  2**2  # build ID
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .gnu.hash     000036d4  0000000000400298  0000000000400298  00000280  2**3
                  ALLOC, READONLY
  4 .dynsym       0000cb70  0000000000403970  0000000000403970  00000280  2**3
                  ALLOC, READONLY
  5 .dynstr       00008423  00000000004104e0  00000000004104e0  00000280  2**0
                  ALLOC, READONLY
  6 .gnu.version  000010f4  0000000000418904  0000000000418904  00000280  2**1
                  ALLOC, READONLY
  7 .gnu.version_r 000000b0  00000000004199f8  00000000004199f8  00000280  2**3
                  ALLOC, READONLY
  8 .rela.dyn     000000c0  0000000000419aa8  0000000000419aa8  00000280  2**3
                  ALLOC, READONLY
  9 .rela.plt     00001398  0000000000419b68  0000000000419b68  00000280  2**3
                  ALLOC, READONLY
 10 .init         0000001a  000000000041af00  000000000041af00  00000280  2**2
                  ALLOC, READONLY, CODE
 11 .plt          00000d20  000000000041af20  000000000041af20  00000280  2**4
                  ALLOC, READONLY, CODE
 12 .text         0008b5d2  000000000041bc40  000000000041bc40  00000280  2**4
                  ALLOC, READONLY, CODE
 13 .fini         00000009  00000000004a7214  00000000004a7214  00000280  2**2
                  ALLOC, READONLY, CODE
 14 .rodata       0001cd0e  00000000004a7220  00000000004a7220  00000280  2**5
                  ALLOC, READONLY
 15 .eh_frame_hdr 00003c74  00000000004c3f30  00000000004c3f30  00000280  2**2
                  ALLOC, READONLY
 16 .eh_frame     0001597c  00000000004c7ba8  00000000004c7ba8  00000280  2**3
                  ALLOC, READONLY
 17 .init_array   00000008  00000000006dddf0  00000000006dddf0  00000280  2**3
                  ALLOC
 18 .fini_array   00000008  00000000006dddf8  00000000006dddf8  00000280  2**3
                  ALLOC
 19 .jcr          00000008  00000000006dde00  00000000006dde00  00000280  2**3
                  ALLOC
 20 .dynamic      000001f0  00000000006dde08  00000000006dde08  00000280  2**3
                  ALLOC
 21 .got          00000008  00000000006ddff8  00000000006ddff8  00000280  2**3
                  ALLOC
 22 .got.plt      000006a0  00000000006de000  00000000006de000  00000280  2**3
                  ALLOC
 23 .data         000083f0  00000000006de6a0  00000000006de6a0  00000280  2**5
                  ALLOC
 24 .bss          00005988  00000000006e6aa0  00000000006e6aa0  00000280  2**5
                  ALLOC
 25 .comment      0000002d  0000000000000000  0000000000000000  00000280  2**0
                  CONTENTS, READONLY
 26 .debug_aranges 00001e30  0000000000000000  0000000000000000  000002ad  2**0
                  CONTENTS, READONLY, DEBUGGING
 27 .debug_info   000acd9c  0000000000000000  0000000000000000  000020dd  2**0
                  CONTENTS, READONLY, DEBUGGING
 28 .debug_abbrev 0000abd2  0000000000000000  0000000000000000  000aee79  2**0
                  CONTENTS, READONLY, DEBUGGING
 29 .debug_line   000313e2  0000000000000000  0000000000000000  000b9a4b  2**0
                  CONTENTS, READONLY, DEBUGGING
 30 .debug_str    00015045  0000000000000000  0000000000000000  000eae2d  2**0
                  CONTENTS, READONLY, DEBUGGING
 31 .debug_loc    0010ef66  0000000000000000  0000000000000000  000ffe72  2**0
                  CONTENTS, READONLY, DEBUGGING
 32 .debug_ranges 00014870  0000000000000000  0000000000000000  0020edd8  2**0
                  CONTENTS, READONLY, DEBUGGING
 33 .gdb_index    00025413  0000000000000000  0000000000000000  00223648  2**0
                  CONTENTS, READONLY, DEBUGGING
```

无论是gdb/readelf/objdump/bpftrace都会根据.note.gnu.build-id来查找`/usr/lib/debug/.build-id`目录，所以这是一种标准的放置debuginfo file的机制。

-   .gnu_debuglink section 默认并不会添加到二进制中。

参考：

1.  gdb 的 [Separate-Debug-Files.html](https://sourceware.org/gdb/onlinedocs/gdb/Separate-Debug-Files.html)


## <span class="section-num">4</span> 安装 debuginfo 文件 {#安装-debuginfo-文件}

CentOS/Redhat：

-   6-7版本：只有 debuginfo 包，同时包含 DWARF 调试符号表和源代码；
-   8 开始：debuginfo 和 debugsource 是分开的两个 package ，前者包含 DWARF格式的.debug_XX Sections,后者包含对应的源代码，两者都被安装到`/usr/lib/debug`目录下；
-   需要启用 debuginfo 和 debugsource YUM REPO；
-   Centos 在用 gcc 编译时，默认开启了 -O2 级别优化，所以可能有写源码中的变量不可见，被表示为 &lt;optimized out&gt;;

debuginfo、debugsource 和binary package三者的名称、版本、架构等必须一致：

-   Binary package: packagename-version-release.architecture.rpm
-   Debuginfo package: packagename-debuginfo-version-release.architecture.rpm
-   Debugsource package: packagename-debugsource-version-release.architecture.rpm

gdb 会自动提示有没有找打 debug symbol，同时会提示安装命令：

```shell
$ gdb -q /bin/ls
Reading symbols from /bin/ls...Reading symbols from .gnu_debugdata for /usr/bin/ls...(no debugging symbols found)...done.
(no debugging symbols found)...done.
Missing separate debuginfos, use: dnf debuginfo-install coreutils-8.30-6.el8.x86_64
(gdb)
```

可以使用 debuginfo-install 来根据binary package来安装对应的 debuginfo 包：

```shell
# debuginfo-install zlib-1.2.11-10.el8.x86_64
```

Ubuntu：

-   一般debug symbols包以 -dbgsym 结尾，如 bash-dbgsym；
-   需要启用 dbgsym Package Source Repo，然后才能查询和安装；

<!--listend-->

```shell
root@lima-ebpf-dev:~# apt search bash |grep -B 1 debug

bash-builtins-dbgsym/jammy 5.1-6ubuntu1 amd64
  debug symbols for bash-builtins
--
bash-dbgsym/jammy 5.1-6ubuntu1 amd64
  debug symbols for bash
--
bash-static-dbgsym/jammy 5.1-6ubuntu1 amd64
  debug symbols for bash-static
--
ddd/jammy 1:3.3.12-5.3build1 amd64
  Data Display Debugger, a graphical debugger frontend
--
ondir-dbgsym/jammy 0.2.3+git0.55279f03-1 amd64
  debug symbols for package ondir

root@lima-ebpf-dev:~# apt install bash-dbgsym

root@lima-ebpf-dev:~# gdb /usr/bin/bash
。。。
Reading symbols from /usr/bin/bash...
Reading symbols from /usr/lib/debug/.build-id/33/a5554034feb2af38e8c75872058883b2988bc5.debug...
(gdb)
```


## <span class="section-num">5</span> 查看调试符号表 {#查看调试符号表}


### <span class="section-num">5.1</span> 显示 build-id 和 debuglink {#显示-build-id-和-debuglink}

-   显示 build-id： readelf -n hello
-   显示 debuglink：readelf --string-dump=.gnu_debuglink hello

<!--listend-->

```shell
root@lima-ebpf-dev:~# objcopy --add-gnu-debuglink=hello.debug hello
root@lima-ebpf-dev:~#
root@lima-ebpf-dev:~# readelf --string-dump=.gnu_debuglink hello

String dump of section '.gnu_debuglink':
  [     0]  hello.debug
  [     e]  [j

root@lima-ebpf-dev:~# readelf -n hello

Displaying notes found in: .note.gnu.property
  Owner                Data size        Description
  GNU                  0x00000020       NT_GNU_PROPERTY_TYPE_0
      Properties: x86 feature: IBT, SHSTK
        x86 ISA needed: x86-64-baseline

Displaying notes found in: .note.gnu.build-id
  Owner                Data size        Description
  GNU                  0x00000014       NT_GNU_BUILD_ID (unique build ID bitstring)
    Build ID: ced7aad9174d074c704867b703c014fec94527df

Displaying notes found in: .note.ABI-tag
  Owner                Data size        Description
  GNU                  0x00000010       NT_GNU_ABI_TAG (ABI version tag)
    OS: Linux, ABI: 3.2.0
root@lima-ebpf-dev:~#
```


### <span class="section-num">5.2</span> objdump -h 显示 .debug_XX Sections {#objdump-h-显示-dot-debug-xx-sections}

-   debug 符号表 Section 以 .debug 开头：

<!--listend-->

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


### <span class="section-num">5.3</span> objdump -g/-W 显示 .debug_XX 详情 {#objdump-g-w-显示-dot-debug-xx-详情}

-   直接从文件中读，或者根据debug-link/build-id从系统`/usr/lib/debug`读取 debug 文件。

<!--listend-->

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
    <22>   DW_AT_high_pc     : (data8) 0x2e
    <2a>   DW_AT_stmt_list   : (sec_offset) 0x0
 <1><2e>: Abbrev Number: 1 (DW_TAG_base_type)
    <2f>   DW_AT_byte_size   : (data1) 8
    <30>   DW_AT_encoding    : (data1) 7        (unsigned)
    <31>   DW_AT_name        : (strp) (offset: 0x0): long unsigned int
 <1><35>: Abbrev Number: 1 (DW_TAG_base_type)
    <36>   DW_AT_byte_size   : (data1) 4
    <37>   DW_AT_encoding    : (data1) 7        (unsigned)
    <38>   DW_AT_name        : (strp) (offset: 0x5): unsigned int
 <1><3c>: Abbrev Number: 1 (DW_TAG_base_type)
    <3d>   DW_AT_byte_size   : (data1) 1
    <3e>   DW_AT_encoding    : (data1) 8        (unsigned char)
    <3f>   DW_AT_name        : (strp) (offset: 0xc3): unsigned char
 <1><43>: Abbrev Number: 1 (DW_TAG_base_type)
    <44>   DW_AT_byte_size   : (data1) 2
    <45>   DW_AT_encoding    : (data1) 7        (unsigned)
    <46>   DW_AT_name        : (strp) (offset: 0x12): short unsigned int
 <1><4a>: Abbrev Number: 1 (DW_TAG_base_type)
    <4b>   DW_AT_byte_size   : (data1) 1
    <4c>   DW_AT_encoding    : (data1) 6        (signed char)
    <4d>   DW_AT_name        : (strp) (offset: 0xc5): signed char
 <1><51>: Abbrev Number: 1 (DW_TAG_base_type)
    <52>   DW_AT_byte_size   : (data1) 2
    <53>   DW_AT_encoding    : (data1) 5        (signed)
    <54>   DW_AT_name        : (strp) (offset: 0x25): short int
 <1><58>: Abbrev Number: 3 (DW_TAG_base_type)
    <59>   DW_AT_byte_size   : (data1) 4
    <5a>   DW_AT_encoding    : (data1) 5        (signed)
    <5b>   DW_AT_name        : (string) int
 <1><5f>: Abbrev Number: 1 (DW_TAG_base_type)
    <60>   DW_AT_byte_size   : (data1) 8
    <61>   DW_AT_encoding    : (data1) 5        (signed)
    <62>   DW_AT_name        : (strp) (offset: 0xd1): long int
 <1><66>: Abbrev Number: 1 (DW_TAG_base_type)
    <67>   DW_AT_byte_size   : (data1) 1
    <68>   DW_AT_encoding    : (data1) 6        (signed char)
    <69>   DW_AT_name        : (strp) (offset: 0xcc): char
 <1><6d>: Abbrev Number: 4 (DW_TAG_subprogram)
    <6e>   DW_AT_external    : (flag_present) 1
    <6e>   DW_AT_name        : (strp) (offset: 0xda): main
    <72>   DW_AT_decl_file   : (data1) 1
    <73>   DW_AT_decl_line   : (data1) 6
    <74>   DW_AT_decl_column : (data1) 5
    <75>   DW_AT_prototyped  : (flag_present) 1
    <75>   DW_AT_type        : (ref4) <0x58>, int
    <79>   DW_AT_low_pc      : (addr) 0x1163
    <81>   DW_AT_high_pc     : (data8) 0x14
    <89>   DW_AT_frame_base  : (exprloc) 1 byte block: 9c       (DW_OP_call_frame_cfa)
    <8b>   DW_AT_call_all_tail_calls: (flag_present) 1
 <1><8b>: Abbrev Number: 5 (DW_TAG_subprogram)
    <8c>   DW_AT_external    : (flag_present) 1
    <8c>   DW_AT_name        : (strp) (offset: 0x2f): hello
    <90>   DW_AT_decl_file   : (data1) 1
    <91>   DW_AT_decl_line   : (data1) 3
    <92>   DW_AT_decl_column : (data1) 6
    <93>   DW_AT_prototyped  : (flag_present) 1
    <93>   DW_AT_low_pc      : (addr) 0x1149
    <9b>   DW_AT_high_pc     : (data8) 0x1a
    <a3>   DW_AT_frame_base  : (exprloc) 1 byte block: 9c       (DW_OP_call_frame_cfa)
    <a5>   DW_AT_call_all_tail_calls: (flag_present) 1
 <1><a5>: Abbrev Number: 0

Contents of the .debug_abbrev section (loaded from /usr/lib/debug/hello.debug):

  Number TAG (0x0)
   1      DW_TAG_base_type    [no children]
    DW_AT_byte_size    DW_FORM_data1
    DW_AT_encoding     DW_FORM_data1
    DW_AT_name         DW_FORM_strp
    DW_AT value: 0     DW_FORM value: 0
   2      DW_TAG_compile_unit    [has children]
    DW_AT_producer     DW_FORM_strp
    DW_AT_language     DW_FORM_data1
    DW_AT_name         DW_FORM_line_strp
    DW_AT_comp_dir     DW_FORM_line_strp
    DW_AT_low_pc       DW_FORM_addr
    DW_AT_high_pc      DW_FORM_data8
    DW_AT_stmt_list    DW_FORM_sec_offset
    DW_AT value: 0     DW_FORM value: 0
   3      DW_TAG_base_type    [no children]
    DW_AT_byte_size    DW_FORM_data1
    DW_AT_encoding     DW_FORM_data1
    DW_AT_name         DW_FORM_string
    DW_AT value: 0     DW_FORM value: 0
   4      DW_TAG_subprogram    [no children]
    DW_AT_external     DW_FORM_flag_present
    DW_AT_name         DW_FORM_strp
    DW_AT_decl_file    DW_FORM_data1
    DW_AT_decl_line    DW_FORM_data1
    DW_AT_decl_column  DW_FORM_data1
    DW_AT_prototyped   DW_FORM_flag_present
    DW_AT_type         DW_FORM_ref4
    DW_AT_low_pc       DW_FORM_addr
    DW_AT_high_pc      DW_FORM_data8
    DW_AT_frame_base   DW_FORM_exprloc
    DW_AT_call_all_tail_calls DW_FORM_flag_present
    DW_AT value: 0     DW_FORM value: 0
   5      DW_TAG_subprogram    [no children]
    DW_AT_external     DW_FORM_flag_present
    DW_AT_name         DW_FORM_strp
    DW_AT_decl_file    DW_FORM_data1
    DW_AT_decl_line    DW_FORM_data1
    DW_AT_decl_column  DW_FORM_data1
    DW_AT_prototyped   DW_FORM_flag_present
    DW_AT_low_pc       DW_FORM_addr
    DW_AT_high_pc      DW_FORM_data8
    DW_AT_frame_base   DW_FORM_exprloc
    DW_AT_call_all_tail_calls DW_FORM_flag_present
    DW_AT value: 0     DW_FORM value: 0

Raw dump of debug contents of section .debug_line (loaded from /usr/lib/debug/hello.debug):

  Offset:                      0x0
  Length:                      87
  DWARF Version:               5
  Address size (bytes):        8
  Segment selector (bytes):    0
  Prologue Length:             42
  Minimum Instruction Length:  1
  Maximum Ops per Instruction: 1
  Initial value of 'is_stmt':  1
  Line Base:                   -5
  Line Range:                  14
  Opcode Base:                 13

 Opcodes:
  Opcode 1 has 0 args
  Opcode 2 has 1 arg
  Opcode 3 has 1 arg
  Opcode 4 has 1 arg
  Opcode 5 has 1 arg
  Opcode 6 has 0 args
  Opcode 7 has 0 args
  Opcode 8 has 0 args
  Opcode 9 has 1 arg
  Opcode 10 has 0 args
  Opcode 11 has 0 args
  Opcode 12 has 1 arg

 The Directory Table (offset 0x22, lines 1, columns 1):
  Entry Name
  0     (line_strp)     (offset: 0x7): /root

 The File Name Table (offset 0x2c, lines 2, columns 2):
  Entry Dir     Name
  0     (udata) 0       (line_strp)     (offset: 0x0): test.c
  1     (udata) 0       (line_strp)     (offset: 0x0): test.c

 Line Number Statements:
  [0x00000036]  Set column to 17
  [0x00000038]  Extended opcode 2: set Address to 0x1149
  [0x00000043]  Special opcode 7: advance Address by 0 to 0x1149 and Line by 2 to 3
  [0x00000044]  Set column to 2
  [0x00000046]  Special opcode 118: advance Address by 8 to 0x1151 and Line by 1 to 4
  [0x00000047]  Set column to 1
  [0x00000049]  Special opcode 216: advance Address by 15 to 0x1160 and Line by 1 to 5
  [0x0000004a]  Set column to 16
  [0x0000004c]  Special opcode 48: advance Address by 3 to 0x1163 and Line by 1 to 6
  [0x0000004d]  Set column to 2
  [0x0000004f]  Special opcode 118: advance Address by 8 to 0x116b and Line by 1 to 7
  [0x00000050]  Set column to 9
  [0x00000052]  Special opcode 76: advance Address by 5 to 0x1170 and Line by 1 to 8
  [0x00000053]  Set column to 1
  [0x00000055]  Special opcode 76: advance Address by 5 to 0x1175 and Line by 1 to 9
  [0x00000056]  Advance PC by 2 to 0x1177
  [0x00000058]  Extended opcode 1: End of Sequence


Contents of the .debug_str section (loaded from /usr/lib/debug/hello.debug):

  0x00000000 6c6f6e67 20756e73 69676e65 6420696e long unsigned in
  0x00000010 74007368 6f727420 756e7369 676e6564 t.short unsigned
  0x00000020 20696e74 0073686f 72742069 6e740068  int.short int.h
  0x00000030 656c6c6f 00474e55 20433137 2031312e ello.GNU C17 11.
  0x00000040 332e3020 2d6d7475 6e653d67 656e6572 3.0 -mtune=gener
  0x00000050 6963202d 6d617263 683d7838 362d3634 ic -march=x86-64
  0x00000060 202d6720 2d666173 796e6368 726f6e6f  -g -fasynchrono
  0x00000070 75732d75 6e77696e 642d7461 626c6573 us-unwind-tables
  0x00000080 202d6673 7461636b 2d70726f 74656374  -fstack-protect
  0x00000090 6f722d73 74726f6e 67202d66 73746163 or-strong -fstac
  0x000000a0 6b2d636c 6173682d 70726f74 65637469 k-clash-protecti
  0x000000b0 6f6e202d 6663662d 70726f74 65637469 on -fcf-protecti
  0x000000c0 6f6e0075 6e736967 6e656420 63686172 on.unsigned char
  0x000000d0 006c6f6e 6720696e 74006d61 696e00   .long int.main.

Contents of the .debug_line_str section (loaded from /usr/lib/debug/hello.debug):

  0x00000000 74657374 2e63002f 726f6f74 00       test.c./root.


hello:     file format elf64-x86-64

Contents of the .eh_frame section (loaded from hello):


00000000 0000000000000014 00000000 CIE
  Version:               1
  Augmentation:          "zR"
  Code alignment factor: 1
  Data alignment factor: -8
  Return address column: 16
  Augmentation data:     1b
  DW_CFA_def_cfa: r7 (rsp) ofs 8
  DW_CFA_offset: r16 (rip) at cfa-8
  DW_CFA_nop
  DW_CFA_nop

00000018 0000000000000014 0000001c FDE cie=00000000 pc=0000000000001060..0000000000001086
  DW_CFA_advance_loc: 4 to 0000000000001064
  DW_CFA_undefined: r16 (rip)
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop

00000030 0000000000000024 00000034 FDE cie=00000000 pc=0000000000001020..0000000000001040
  DW_CFA_def_cfa_offset: 16
  DW_CFA_advance_loc: 6 to 0000000000001026
  DW_CFA_def_cfa_offset: 24
  DW_CFA_advance_loc: 10 to 0000000000001030
  DW_CFA_def_cfa_expression (DW_OP_breg7 (rsp): 8; DW_OP_breg16 (rip): 0; DW_OP_lit15; DW_OP_and; DW_OP_lit10; DW_OP_ge; DW_OP_lit3; DW_OP_shl; DW_OP_plus)
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop

00000058 0000000000000014 0000005c FDE cie=00000000 pc=0000000000001040..0000000000001050
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop

00000070 0000000000000014 00000074 FDE cie=00000000 pc=0000000000001050..0000000000001060
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop

00000088 000000000000001c 0000008c FDE cie=00000000 pc=0000000000001149..0000000000001163
  DW_CFA_advance_loc: 5 to 000000000000114e
  DW_CFA_def_cfa_offset: 16
  DW_CFA_offset: r6 (rbp) at cfa-16
  DW_CFA_advance_loc: 3 to 0000000000001151
  DW_CFA_def_cfa_register: r6 (rbp)
  DW_CFA_advance_loc: 17 to 0000000000001162
  DW_CFA_def_cfa: r7 (rsp) ofs 8
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop

000000a8 000000000000001c 000000ac FDE cie=00000000 pc=0000000000001163..0000000000001177
  DW_CFA_advance_loc: 5 to 0000000000001168
  DW_CFA_def_cfa_offset: 16
  DW_CFA_offset: r6 (rbp) at cfa-16
  DW_CFA_advance_loc: 3 to 000000000000116b
  DW_CFA_def_cfa_register: r6 (rbp)
  DW_CFA_advance_loc: 11 to 0000000000001176
  DW_CFA_def_cfa: r7 (rsp) ofs 8
  DW_CFA_nop
  DW_CFA_nop
  DW_CFA_nop

000000c8 ZERO terminator


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


## <span class="section-num">6</span> kernel-debuginfo {#kernel-debuginfo}

内核的调试符号表也被安装到`/usr/lib/debug`目录下：

-   /usr/lib/debug/lib/modules 对应各种内核模块的符号文件；

<!--listend-->

```shell
[root@h33o09169.sqa.nu8 /etc/yum.repos.d]
#rpm -ql kernel-debuginfo-4.19.91-013.ali4000.alios7.x86_64|head
/usr/lib/debug
/usr/lib/debug/.build-id
/usr/lib/debug/.build-id/00
/usr/lib/debug/.build-id/00/0fc26ae19e5ae8e6ecdd110b0dbd24741726d0
/usr/lib/debug/.build-id/00/0fc26ae19e5ae8e6ecdd110b0dbd24741726d0.debug
/usr/lib/debug/.build-id/00/419d56d31f38d06ad54ed389ac5bc884ca543a
/usr/lib/debug/.build-id/00/419d56d31f38d06ad54ed389ac5bc884ca543a.debug
/usr/lib/debug/.build-id/00/56eaf46acde0ab68bdb4c5b1c5147aa0fff269
/usr/lib/debug/.build-id/00/56eaf46acde0ab68bdb4c5b1c5147aa0fff269.debug
/usr/lib/debug/.build-id/00/75d2fadb1b5c88af9d0ad121b3f4705f5c328c
。。。
/usr/lib/debug/lib
/usr/lib/debug/lib/modules
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/kernel
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/kernel/arch
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/kernel/arch/x86
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/kernel/arch/x86/crypto
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/kernel/arch/x86/crypto/aesni-intel.ko.debug
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/kernel/arch/x86/crypto/blowfish-x86_64.ko.debug
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/kernel/arch/x86/crypto/camellia-aesni-avx-x86_64.ko.debug
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/kernel/arch/x86/crypto/camellia-aesni-avx2.ko.debug
。。。
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/kernel/net/wireless/lib80211.ko.debug
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/kernel/net/xfrm
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/kernel/net/xfrm/xfrm_ipcomp.ko.debug
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/kernel/virt
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/kernel/virt/lib
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/kernel/virt/lib/irqbypass.ko.debug
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/vdso
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/vdso/.build-id
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/vdso/.build-id/81
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/vdso/.build-id/81/2688c147f9806c2f13de03a3d7593deccd4c91.debug.debug
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/vdso/.build-id/9c
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/vdso/.build-id/9c/855176d6cfba59c6ae81c0b5d7bf01cc34c385.debug.debug
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/vdso/vdso32.so.debug
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/vdso/vdso64.so.debug
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/vmlinux
/usr/lib/debug/usr
/usr/lib/debug/usr/src
/usr/lib/debug/usr/src/kernels
/usr/lib/debug/usr/src/kernels/4.19.91-013.ali4000.alios7.x86_64
/usr/lib/debug/usr/src/kernels/4.19.91-013.ali4000.alios7.x86_64/scripts
/usr/lib/debug/usr/src/kernels/4.19.91-013.ali4000.alios7.x86_64/scripts/asn1_compiler.debug
/usr/lib/debug/usr/src/kernels/4.19.91-013.ali4000.alios7.x86_64/scripts/basic
/usr/lib/debug/usr/src/kernels/4.19.91-013.ali4000.alios7.x86_64/scripts/basic/fixdep.debug
/usr/lib/debug/usr/src/kernels/4.19.91-013.ali4000.alios7.x86_64/scripts/bin2c.debug
/usr/lib/debug/usr/src/kernels/4.19.91-013.ali4000.alios7.x86_64/scripts/conmakehash.debug
```

`/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/vmlinux` 对应完整的内核 elf 文件：

-   包含各种.debug_XX调试符号表；
-   也包含未 strip 的 .symtab 符号表；

<!--listend-->

```shell
# file /usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/vmlinux
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/vmlinux: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, BuildID[sha1]=854b5a499ad2fe963e37462d882866a60cf5b35a, not stripped

# objdump -h /usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/vmlinux

/usr/lib/debug/lib/modules/4.19.91-013.ali4000.alios7.x86_64/vmlinux:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         00c03000  ffffffff81000000  0000000001000000  00200000  2**12
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .noinstr.text 00000097  ffffffff81c03000  0000000001c03000  00e03000  2**4
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  2 .notes        000001bc  ffffffff81c03098  0000000001c03098  00e03098  2**2
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
  3 __ex_table    00001c44  ffffffff81c03260  0000000001c03260  00e03260  2**2
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
  4 .rodata       0038ac62  ffffffff81e00000  0000000001e00000  01000000  2**7
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
  5 .orc_unwind_ip 0015b6ac  ffffffff8218ac64  000000000218ac64  0138ac64  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
  6 .orc_unwind   00209202  ffffffff822e6310  00000000022e6310  014e6310  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  7 .orc_lookup   000300c4  ffffffff824ef514  00000000024ef514  016ef514  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  8 .pci_fixup    00002c20  ffffffff8251f5e0  000000000251f5e0  0171f5e0  2**4
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
  9 __ksymtab     00009a30  ffffffff82522200  0000000002522200  01722200  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
 10 __ksymtab_gpl 00009090  ffffffff8252bc30  000000000252bc30  0172bc30  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
 11 __kcrctab     00004d18  ffffffff82534cc0  0000000002534cc0  01734cc0  2**2
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
 12 __kcrctab_gpl 00004848  ffffffff825399d8  00000000025399d8  017399d8  2**2
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
 13 __ksymtab_strings 0002d743  ffffffff8253e220  000000000253e220  0173e220  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
 14 __param       00002ee0  ffffffff8256b968  000000000256b968  0176b968  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
 15 __modver      000007b8  ffffffff8256e848  000000000256e848  0176e848  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
 16 .data         001a43c0  ffffffff82600000  0000000002600000  01800000  2**13
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
 17 __bug_table   00012ea0  ffffffff827a43c0  00000000027a43c0  019a43c0  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
 18 .vvar         00001000  ffffffff827b8000  00000000027b8000  019b8000  2**4
                  CONTENTS, ALLOC, LOAD, DATA
 19 .data..percpu 00026000  0000000000000000  00000000027b9000  01a00000  2**12
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
 20 .init.text    000776a5  ffffffff827df000  00000000027df000  01bdf000  2**4
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
 21 .altinstr_aux 00000aed  ffffffff828566a5  00000000028566a5  01c566a5  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
 22 .init.data    000eab98  ffffffff82858000  0000000002858000  01c58000  2**13
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
 23 .x86_cpu_dev.init 00000028  ffffffff82942b98  0000000002942b98  01d42b98  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
 24 .parainstructions 000202ec  ffffffff82942bc0  0000000002942bc0  01d42bc0  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
 25 .altinstructions 00004877  ffffffff82962eb0  0000000002962eb0  01d62eb0  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
 26 .altinstr_replacement 00001281  ffffffff82967727  0000000002967727  01d67727  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
 27 .iommu_table  000000f0  ffffffff829689a8  00000000029689a8  01d689a8  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
 28 .apicdrivers  00000028  ffffffff82968a98  0000000002968a98  01d68a98  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, DATA
 29 .exit.text    000017d0  ffffffff82968ac0  0000000002968ac0  01d68ac0  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
 30 .smp_locks    00009000  ffffffff8296b000  000000000296b000  01d6b000  2**2
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
 31 .data_nosave  00001000  ffffffff82974000  0000000002974000  01d74000  2**2
                  CONTENTS, ALLOC, LOAD, DATA
 32 .bss          0248b000  ffffffff82975000  0000000002975000  01d75000  2**12
                  ALLOC
 33 .brk          0002c000  ffffffff84e00000  0000000004e00000  01d75000  2**0
                  ALLOC
 34 .comment      0000002c  0000000000000000  0000000000000000  01d75000  2**0
                  CONTENTS, READONLY
 35 .debug_aranges 00027cb0  0000000000000000  0000000000000000  01d75030  2**4
                  CONTENTS, RELOC, READONLY, DEBUGGING
 36 .debug_info   0b5eb3dd  0000000000000000  0000000000000000  01d9cce0  2**0
                  CONTENTS, RELOC, READONLY, DEBUGGING
 37 .debug_abbrev 003b1205  0000000000000000  0000000000000000  0d3880bd  2**0
                  CONTENTS, READONLY, DEBUGGING
 38 .debug_line   00a90e2e  0000000000000000  0000000000000000  0d7392c2  2**0
                  CONTENTS, RELOC, READONLY, DEBUGGING
 39 .debug_frame  0024f7a8  0000000000000000  0000000000000000  0e1ca0f0  2**3
                  CONTENTS, RELOC, READONLY, DEBUGGING
 40 .debug_str    002f3d57  0000000000000000  0000000000000000  0e419898  2**0
                  CONTENTS, READONLY, DEBUGGING
 41 .debug_loc    00c29a65  0000000000000000  0000000000000000  0e70d5ef  2**0
                  CONTENTS, RELOC, READONLY, DEBUGGING
 42 .debug_ranges 0050db50  0000000000000000  0000000000000000  0f337060  2**4
                  CONTENTS, RELOC, READONLY, DEBUGGING
 43 .gdb_index    004f73ea  0000000000000000  0000000000000000  0f844bb0  2**0
                  CONTENTS, READONLY, DEBUGGING
```
