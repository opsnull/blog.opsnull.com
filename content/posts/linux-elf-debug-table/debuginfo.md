---
title: "Linux elf 调试符号表（.debug_XX)"
author: ["opsnull"]
date: 2023-08-06T00:00:00+08:00
lastmod: 2023-08-06T21:24:41+08:00
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

由于 debuginfo 通常比较大，而且一般只在 debug 时使用。所以编译器默认不生成它们。在编译二进制时，通过指定 -g 选项，可以在生成的二进制中包含.debug_XX开头的调试符号表，如 .debug_info, .debug_abbrev, .debug_line,
.debug_frame, .debug_str, .debug_loc 等，它们符合`DWARF`格式标准；

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
...
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
...
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
...
Displaying notes found in: .note.gnu.build-id
  Owner                Data size        Description
  GNU                  0x00000014       NT_GNU_BUILD_ID (unique build ID bitstring)
    Build ID: ced7aad9174d074c704867b703c014fec94527df
...
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
...
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

-   直接从文件中读或者根据debug-link/build-id从系统`/usr/lib/debug`读取 debug 文件。

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
...
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
...

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
...

Contents of the .debug_str section (loaded from /usr/lib/debug/hello.debug):

  0x00000000 6c6f6e67 20756e73 69676e65 6420696e long unsigned in
  0x00000010 74007368 6f727420 756e7369 676e6564 t.short unsigned
  0x00000020 20696e74 0073686f 72742069 6e740068  int.short int.h
  0x00000030 656c6c6f 00474e55 20433137 2031312e ello.GNU C17 11.
  0x00000040 332e3020 2d6d7475 6e653d67 656e6572 3.0 -mtune=gener
...
Contents of the .debug_line_str section (loaded from /usr/lib/debug/hello.debug):

  0x00000000 74657374 2e63002f 726f6f74 00       test.c./root.


hello:     file format elf64-x86-64

Contents of the .eh_frame section (loaded from hello):

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


## <span class="section-num">6</span> kernel-debuginfo {#kernel-debuginfo}

内核的调试符号表也被安装到`/usr/lib/debug`目录下：

-   /usr/lib/debug/lib/modules 对应各种内核模块的符号文件；

<!--listend-->

```shell
# rpm -ql kernel-debuginfo-4.19.91-013.ali4000.os7.x86_64|head
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
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/kernel
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/kernel/arch
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/kernel/arch/x86
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/kernel/arch/x86/crypto
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/kernel/arch/x86/crypto/aesni-intel.ko.debug
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/kernel/arch/x86/crypto/blowfish-x86_64.ko.debug
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/kernel/arch/x86/crypto/camellia-aesni-avx-x86_64.ko.debug
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/kernel/arch/x86/crypto/camellia-aesni-avx2.ko.debug
。。。
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/kernel/net/wireless/lib80211.ko.debug
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/kernel/net/xfrm
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/kernel/net/xfrm/xfrm_ipcomp.ko.debug
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/kernel/virt
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/kernel/virt/lib
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/kernel/virt/lib/irqbypass.ko.debug
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/vdso
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/vdso/.build-id
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/vdso/.build-id/81
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/vdso/.build-id/81/2688c147f9806c2f13de03a3d7593deccd4c91.debug.debug
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/vdso/.build-id/9c
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/vdso/.build-id/9c/855176d6cfba59c6ae81c0b5d7bf01cc34c385.debug.debug
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/vdso/vdso32.so.debug
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/vdso/vdso64.so.debug
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/vmlinux
/usr/lib/debug/usr
/usr/lib/debug/usr/src
/usr/lib/debug/usr/src/kernels
/usr/lib/debug/usr/src/kernels/4.19.91-013.ali4000.os7.x86_64
/usr/lib/debug/usr/src/kernels/4.19.91-013.ali4000.os7.x86_64/scripts
/usr/lib/debug/usr/src/kernels/4.19.91-013.ali4000.os7.x86_64/scripts/asn1_compiler.debug
/usr/lib/debug/usr/src/kernels/4.19.91-013.ali4000.os7.x86_64/scripts/basic
/usr/lib/debug/usr/src/kernels/4.19.91-013.ali4000.os7.x86_64/scripts/basic/fixdep.debug
/usr/lib/debug/usr/src/kernels/4.19.91-013.ali4000.os7.x86_64/scripts/bin2c.debug
/usr/lib/debug/usr/src/kernels/4.19.91-013.ali4000.os7.x86_64/scripts/conmakehash.debug
```

`/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/vmlinux` 对应完整的内核 elf 文件：

-   包含各种.debug_XX调试符号表；
-   也包含未 strip 的 .symtab 符号表；

<!--listend-->

```shell
# file /usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/vmlinux
/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/vmlinux: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, BuildID[sha1]=854b5a499ad2fe963e37462d882866a60cf5b35a, not stripped

# objdump -h /usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/vmlinux

/usr/lib/debug/lib/modules/4.19.91-013.ali4000.os7.x86_64/vmlinux:     file format elf64-x86-64

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
...
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
