---
title: "Linux elf 符号表（symtab）"
author: ["opsnull"]
date: 2023-08-06T00:00:00+08:00
lastmod: 2023-08-07T22:22:12+08:00
tags: ["linux", "elf", "debug"]
categories: ["debug"]
draft: false
series: ["elf-debug"]
series_order: 1
---

介绍Linux elf二进制文件的符号表（ symtab ）生成和管理机制。

<!--more-->

Linux 使用 ELF 格式类保存可执行二进制文件、共享库文件和 debuginfo文件。

ELF 使用两个 Sections 来表示 symbol table：

-   .dynsym：dynamic symbols, which can be used by another program;
-   .symtab：local symbols used by the binary itself only;

.symtab 保存了程序中`标识符和内存地址的对应关系`，在进行`uprobe/stack unwinding/debug`时使用。


## <span class="section-num">1</span> 生成符号表 {#生成符号表}

gcc 编译时默认会生成这两个符号表，`file`命令显示 `not stripped`

-   `-g` 表示同时生成调试符号表（以`.debug`开头的 Sections ， `DWARF`格式），file 会显示 `with debug_info`;

<!--listend-->

```shell
root@lima-ebpf-dev:~# gcc -g test.c -o hello
root@lima-ebpf-dev:~# file hello
hello: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=a513ea9e31dc7b8ffbf423f56a04fbff6c0e99a9, for GNU/Linux 3.2.0, with debug_info, not stripped
```


## <span class="section-num">2</span> 查看符号表 {#查看符号表}

使用`readelf -s/objdump -t`命令显示符号表内容：

```shell
root@lima-ebpf-dev:~# readelf -S hello |grep sym
  [ 6] .dynsym           DYNSYM           00000000000003d8  000003d8
  [35] .symtab           SYMTAB           0000000000000000  000032d0

root@lima-ebpf-dev:~# readelf -s hello|grep Symbo
Symbol table '.dynsym' contains 7 entries:
Symbol table '.symtab' contains 37 entries:

# 符号表中有 hello 函数的地址
# readelf -s hello|grep hello
    52: 000000000040057d    26 FUNC    GLOBAL DEFAULT   13 hello

# readelf -s hello

Symbol table '.dynsym' contains 5 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@GLIBC_2.2.5 (2)
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@GLIBC_2.2.5 (2)
     3: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     4: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND sleep@GLIBC_2.2.5 (2)

Symbol table '.symtab' contains 65 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000400238     0 SECTION LOCAL  DEFAULT    1
     2: 0000000000400254     0 SECTION LOCAL  DEFAULT    2
     3: 0000000000400274     0 SECTION LOCAL  DEFAULT    3
     4: 0000000000400298     0 SECTION LOCAL  DEFAULT    4
     5: 00000000004002b8     0 SECTION LOCAL  DEFAULT    5
     6: 0000000000400330     0 SECTION LOCAL  DEFAULT    6
     7: 0000000000400374     0 SECTION LOCAL  DEFAULT    7
     8: 0000000000400380     0 SECTION LOCAL  DEFAULT    8
     9: 00000000004003a0     0 SECTION LOCAL  DEFAULT    9
    10: 00000000004003b8     0 SECTION LOCAL  DEFAULT   10
    11: 0000000000400418     0 SECTION LOCAL  DEFAULT   11
    12: 0000000000400440     0 SECTION LOCAL  DEFAULT   12
    13: 0000000000400490     0 SECTION LOCAL  DEFAULT   13
    14: 0000000000400624     0 SECTION LOCAL  DEFAULT   14
    15: 0000000000400630     0 SECTION LOCAL  DEFAULT   15
    16: 0000000000400650     0 SECTION LOCAL  DEFAULT   16
    17: 0000000000400690     0 SECTION LOCAL  DEFAULT   17
    18: 0000000000600e10     0 SECTION LOCAL  DEFAULT   18
    19: 0000000000600e18     0 SECTION LOCAL  DEFAULT   19
    20: 0000000000600e20     0 SECTION LOCAL  DEFAULT   20
    21: 0000000000600e28     0 SECTION LOCAL  DEFAULT   21
    22: 0000000000600ff8     0 SECTION LOCAL  DEFAULT   22
    23: 0000000000601000     0 SECTION LOCAL  DEFAULT   23
    24: 0000000000601038     0 SECTION LOCAL  DEFAULT   24
    25: 000000000060103c     0 SECTION LOCAL  DEFAULT   25
    26: 0000000000000000     0 SECTION LOCAL  DEFAULT   26
    27: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    28: 0000000000600e20     0 OBJECT  LOCAL  DEFAULT   20 __JCR_LIST__
    29: 00000000004004c0     0 FUNC    LOCAL  DEFAULT   13 deregister_tm_clones
    30: 00000000004004f0     0 FUNC    LOCAL  DEFAULT   13 register_tm_clones
    31: 0000000000400530     0 FUNC    LOCAL  DEFAULT   13 __do_global_dtors_aux
    32: 000000000060103c     1 OBJECT  LOCAL  DEFAULT   25 completed.6355
    33: 0000000000600e18     0 OBJECT  LOCAL  DEFAULT   19 __do_global_dtors_aux_fin
    34: 0000000000400550     0 FUNC    LOCAL  DEFAULT   13 frame_dummy
    35: 0000000000600e10     0 OBJECT  LOCAL  DEFAULT   18 __frame_dummy_init_array_
    36: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS test.c
    37: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    38: 00000000004007a0     0 OBJECT  LOCAL  DEFAULT   17 __FRAME_END__
    39: 0000000000600e20     0 OBJECT  LOCAL  DEFAULT   20 __JCR_END__
    40: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS
    41: 0000000000600e18     0 NOTYPE  LOCAL  DEFAULT   18 __init_array_end
    42: 0000000000600e28     0 OBJECT  LOCAL  DEFAULT   21 _DYNAMIC
    43: 0000000000600e10     0 NOTYPE  LOCAL  DEFAULT   18 __init_array_start
    44: 0000000000400650     0 NOTYPE  LOCAL  DEFAULT   16 __GNU_EH_FRAME_HDR
    45: 0000000000601000     0 OBJECT  LOCAL  DEFAULT   23 _GLOBAL_OFFSET_TABLE_
    46: 0000000000400620     2 FUNC    GLOBAL DEFAULT   13 __libc_csu_fini
    47: 0000000000601038     0 NOTYPE  WEAK   DEFAULT   24 data_start
    48: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@@GLIBC_2.2.5
    49: 000000000060103c     0 NOTYPE  GLOBAL DEFAULT   24 _edata
    50: 0000000000400624     0 FUNC    GLOBAL DEFAULT   14 _fini
    51: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@@GLIBC_
    52: 000000000040057d    26 FUNC    GLOBAL DEFAULT   13 hello
    53: 0000000000601038     0 NOTYPE  GLOBAL DEFAULT   24 __data_start
    54: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
    55: 0000000000400638     0 OBJECT  GLOBAL HIDDEN    15 __dso_handle
    56: 0000000000400630     4 OBJECT  GLOBAL DEFAULT   15 _IO_stdin_used
    57: 00000000004005b0   101 FUNC    GLOBAL DEFAULT   13 __libc_csu_init
    58: 0000000000601040     0 NOTYPE  GLOBAL DEFAULT   25 _end
    59: 0000000000400490     0 FUNC    GLOBAL DEFAULT   13 _start
    60: 000000000060103c     0 NOTYPE  GLOBAL DEFAULT   25 __bss_start
    61: 0000000000400597    16 FUNC    GLOBAL DEFAULT   13 main
    62: 0000000000601040     0 OBJECT  GLOBAL HIDDEN    24 __TMC_END__
    63: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND sleep@@GLIBC_2.2.5
    64: 0000000000400418     0 FUNC    GLOBAL DEFAULT   11 _init
```


## <span class="section-num">3</span> 删除符号表 {#删除符号表}

使用`strip -s/--strip-all`命令同时删除二进制文件中的`.symtab`和 `.debug_XX` 调试符号表内容，file 命令显示
`stripped` ；

-   `.dynsym` 是二进制给外界的符号表，不能删除；

<!--listend-->

```shell
# strip -s hello

# file hello
hello: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, BuildID[sha1]=86296aac1aa4149c55d40ced40c4ca8fbe3b17a9, stripped

# readelf -s hello|grep Symbo
Symbol table '.dynsym' contains 5 entries:

# readelf -s hello|grep hello
```

删除`.symtab`符号表后，`uprobe`将失败：

-   kprobe/uprobe 使用.symtab来查找符号名称（变量、函数、类型）和内存地址的关系，不依赖 debug symbol table；

<!--listend-->

```shell
#bpftrace -e 'uprobe:./hello:hello {printf("%s",ustack)}'  -c ./hello
Attaching 1 probe...
hello world

        hello+0
        __libc_start_main+245

#strip -s hello

#bpftrace -e 'uprobe:./hello:hello {printf("%s",ustack)}'  -c ./hello
No probes to attach
```


## <span class="section-num">4</span> 生成 debuginfo 文件 {#生成-debuginfo-文件}

使用`objcopy --only-keep-debug hello hello.debug`生成的 debuginfo 文件中包含.symtab和各种.debug_XX调试符号表：

```shell
root@lima-ebpf-dev:~# readelf -S hello.debug  |grep -E 'dynsym|symta'
readelf: Error: Unable to find program interpreter name
  [ 6] .dynsym           NOBITS           00000000000003d8  000003ac
  [34] .symtab           SYMTAB           0000000000000000  00000658
```


## <span class="section-num">5</span> 从 debuginfo 文件合并符号表 {#从-debuginfo-文件合并符号表}

和 strip 相反的工具是 elfutils 包提供的`eu-unstrip`工具，它可以将exec binary和 debuginfo 文件合并到一起，形成一个包含符号表和调试符号表的exec binary文件：

```shell
root@lima-ebpf-dev:~# eu-unstrip -f /usr/bin/bash /usr/lib/debug/.build-id/33/a5554034feb2af38e8c75872058883b2988bc5.debug -o /tmp/bash
root@lima-ebpf-dev:~#
root@lima-ebpf-dev:~# readelf -S /tmp/bash |grep sym
  [ 6] .dynsym           DYNSYM           0000000000004f68  00004f68
  [37] .symtab           SYMTAB           0000000000000000  002fb538
root@lima-ebpf-dev:~# readelf -S /tmp/bash |grep -E 'sym|\.debug_'
  [ 6] .dynsym           DYNSYM           0000000000004f68  00004f68
  [29] .debug_aranges    PROGBITS         0000000000000000  00154678
  [30] .debug_info       PROGBITS         0000000000000000  00154820
  [31] .debug_abbrev     PROGBITS         0000000000000000  00215af0
  [32] .debug_line       PROGBITS         0000000000000000  0021ad30
  [33] .debug_str        PROGBITS         0000000000000000  0027ad40
  [34] .debug_line_str   PROGBITS         0000000000000000  002840b0
  [35] .debug_loclists   PROGBITS         0000000000000000  002847a0
  [36] .debug_rnglists   PROGBITS         0000000000000000  002ec800
  [37] .symtab           SYMTAB           0000000000000000  002fb538
root@lima-ebpf-dev:~#
```
