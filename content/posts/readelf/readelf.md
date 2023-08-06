---
title: "readelf"
author: ["opsnull"]
date: 2023-08-06T00:00:00+08:00
lastmod: 2023-08-06T21:43:36+08:00
tags: ["linux", "elf", "debug", "tools"]
categories: ["debug", "tools"]
draft: false
series: ["elf-debug"]
series_order: 3
---

介绍 readelf 命令的功能用法。

<!--more-->

readelf 和 objdump 相比：

1.  objdump 可以对二进制文件根据调试符号表进行反汇编，但是 readelf 不行；
2.  两者都会使用 build-id 和 gnu debuglink 机制从`/usr/lib/debug`查找当前二进制的 debuginfo 文件，然后显示其内容；

添加`-W/--wide`选项，可以使输出宽度超过 80 字符。


## <span class="section-num">1</span> 显示 ELF header {#显示-elf-header}

`readelf -h/--file-headers`: 显示 ELF file header

```shell
root@lima-ebpf-dev:~# readelf -h hello
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Position-Independent Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x1060
  Start of program headers:          64 (bytes into file)
  Start of section headers:          14720 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         13
  Size of section headers:           64 (bytes)
  Number of section headers:         37
  Section header string table index: 36
```


## <span class="section-num">2</span> 显示 program headers 列表（Segments）： {#显示-program-headers-列表-segments}

readelf -l/--program-headers/--segments: 显示program headers的内容（也称为 Segments）：

-   PHDR：
-   INTERP：/lib64/ld-linux-x86-64.so.2
-   LOAD
-   DYNAMIC；
-   NOTE
-   GNU_PROPERTY
-   GNU_EH_FRAME: 在C++ exception resolution场景使用。
-   GNU_STACK
-   GNU_RELRO

<!--listend-->

```shell
root@lima-ebpf-dev:~# readelf -l -W hello # -W/--wide 表示输出宽度可以超过 80 字符。

Elf file type is DYN (Position-Independent Executable file)
Entry point 0x1060
There are 13 program headers, starting at offset 64

Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
  PHDR           0x000040 0x0000000000000040 0x0000000000000040 0x0002d8 0x0002d8 R   0x8
  INTERP         0x000318 0x0000000000000318 0x0000000000000318 0x00001c 0x00001c R   0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x000000 0x0000000000000000 0x0000000000000000 0x000628 0x000628 R   0x1000
  LOAD           0x001000 0x0000000000001000 0x0000000000001000 0x000185 0x000185 R E 0x1000
  LOAD           0x002000 0x0000000000002000 0x0000000000002000 0x000114 0x000114 R   0x1000
  LOAD           0x002db8 0x0000000000003db8 0x0000000000003db8 0x000258 0x000260 RW  0x1000
  DYNAMIC        0x002dc8 0x0000000000003dc8 0x0000000000003dc8 0x0001f0 0x0001f0 RW  0x8
  NOTE           0x000338 0x0000000000000338 0x0000000000000338 0x000030 0x000030 R   0x8
  NOTE           0x000368 0x0000000000000368 0x0000000000000368 0x000044 0x000044 R   0x4
  GNU_PROPERTY   0x000338 0x0000000000000338 0x0000000000000338 0x000030 0x000030 R   0x8
  GNU_EH_FRAME   0x00200c 0x000000000000200c 0x000000000000200c 0x00003c 0x00003c R   0x4
  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0x10
  GNU_RELRO      0x002db8 0x0000000000003db8 0x0000000000003db8 0x000248 0x000248 R   0x1

 Section to Segment mapping:
  Segment Sections...
   00
   01     .interp
   02     .interp .note.gnu.property .note.gnu.build-id .note.ABI-tag .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt
   03     .init .plt .plt.got .plt.sec .text .fini
   04     .rodata .eh_frame_hdr .eh_frame
   05     .init_array .fini_array .dynamic .got .data .bss
   06     .dynamic
   07     .note.gnu.property
   08     .note.gnu.build-id .note.ABI-tag
   09     .note.gnu.property
   10     .eh_frame_hdr
   11
   12     .init_array .fini_array .dynamic .got
root@lima-ebpf-dev:~#
```


## <span class="section-num">3</span> 显示所有 Sections {#显示所有-sections}

`readelf -S/--sections/--section-headers` ：显示所有section headers列表；

-   带有 S 标识的表示是字符串 Sections，如 .comment;
-   使用 -W 来显示超过 80 字符的宽度；
-   只显示当前文件，而不查找 debuginfo 文件；

<!--listend-->

```shell
root@lima-ebpf-dev:~# readelf -S -W hello
There are 37 section headers, starting at offset 0x3980:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .interp           PROGBITS        0000000000000318 000318 00001c 00   A  0   0  1
  [ 2] .note.gnu.property NOTE            0000000000000338 000338 000030 00   A  0   0  8
  [ 3] .note.gnu.build-id NOTE            0000000000000368 000368 000024 00   A  0   0  4
  [ 4] .note.ABI-tag     NOTE            000000000000038c 00038c 000020 00   A  0   0  4
  [ 5] .gnu.hash         GNU_HASH        00000000000003b0 0003b0 000024 00   A  6   0  8
  [ 6] .dynsym           DYNSYM          00000000000003d8 0003d8 0000a8 18   A  7   1  8
  [ 7] .dynstr           STRTAB          0000000000000480 000480 00008d 00   A  0   0  1
  [ 8] .gnu.version      VERSYM          000000000000050e 00050e 00000e 02   A  6   0  2
  [ 9] .gnu.version_r    VERNEED         0000000000000520 000520 000030 00   A  7   1  8
  [10] .rela.dyn         RELA            0000000000000550 000550 0000c0 18   A  6   0  8
  [11] .rela.plt         RELA            0000000000000610 000610 000018 18  AI  6  24  8
  [12] .init             PROGBITS        0000000000001000 001000 00001b 00  AX  0   0  4
  [13] .plt              PROGBITS        0000000000001020 001020 000020 10  AX  0   0 16
  [14] .plt.got          PROGBITS        0000000000001040 001040 000010 10  AX  0   0 16
  [15] .plt.sec          PROGBITS        0000000000001050 001050 000010 10  AX  0   0 16
  [16] .text             PROGBITS        0000000000001060 001060 000117 00  AX  0   0 16
  [17] .fini             PROGBITS        0000000000001178 001178 00000d 00  AX  0   0  4
  [18] .rodata           PROGBITS        0000000000002000 002000 00000b 00   A  0   0  4
  [19] .eh_frame_hdr     PROGBITS        000000000000200c 00200c 00003c 00   A  0   0  4
  [20] .eh_frame         PROGBITS        0000000000002048 002048 0000cc 00   A  0   0  8
  [21] .init_array       INIT_ARRAY      0000000000003db8 002db8 000008 08  WA  0   0  8
  [22] .fini_array       FINI_ARRAY      0000000000003dc0 002dc0 000008 08  WA  0   0  8
  [23] .dynamic          DYNAMIC         0000000000003dc8 002dc8 0001f0 10  WA  7   0  8
  [24] .got              PROGBITS        0000000000003fb8 002fb8 000048 08  WA  0   0  8
  [25] .data             PROGBITS        0000000000004000 003000 000010 00  WA  0   0  8
  [26] .bss              NOBITS          0000000000004010 003010 000008 00  WA  0   0  1
  [27] .comment          PROGBITS        0000000000000000 003010 00002d 01  MS  0   0  1
  [28] .debug_aranges    PROGBITS        0000000000000000 00303d 000030 00      0   0  1
  [29] .debug_info       PROGBITS        0000000000000000 00306d 0000a6 00      0   0  1
  [30] .debug_abbrev     PROGBITS        0000000000000000 003113 00005e 00      0   0  1
  [31] .debug_line       PROGBITS        0000000000000000 003171 00005b 00      0   0  1
  [32] .debug_str        PROGBITS        0000000000000000 0031cc 0000df 01  MS  0   0  1
  [33] .debug_line_str   PROGBITS        0000000000000000 0032ab 00000d 01  MS  0   0  1
  [34] .symtab           SYMTAB          0000000000000000 0032b8 000378 18     35  18  8
  [35] .strtab           STRTAB          0000000000000000 003630 0001e0 00      0   0  1
  [36] .shstrtab         STRTAB          0000000000000000 003810 00016a 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  D (mbind), l (large), p (processor specific)
root@lima-ebpf-dev:~#
```

readelf -e/--headers：等效为-h -l -S，即同时显示 ELF/Program/Section headers的内容。


## <span class="section-num">4</span> 显示符号表（.dynsym &amp; .symtab) {#显示符号表-dot-dynsym-and-dot-symtab}

readelf -s/--syms/--symbols：显示符号表（symbol table）：'.dynsym'和 '.symtab'：

-   只显示当前文件，而不查找 debuginfo 文件；

<!--listend-->

```shell
root@lima-ebpf-dev:~# readelf -s hello

Symbol table '.dynsym' contains 7 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND _[...]@GLIBC_2.34 (2)
     2: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterT[...]
     3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@GLIBC_2.2.5 (3)
     4: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     5: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMC[...]
     6: 0000000000000000     0 FUNC    WEAK   DEFAULT  UND [...]@GLIBC_2.2.5 (3)

Symbol table '.symtab' contains 37 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS Scrt1.o
     2: 000000000000038c    32 OBJECT  LOCAL  DEFAULT    4 __abi_tag
     3: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
     4: 0000000000001090     0 FUNC    LOCAL  DEFAULT   16 deregister_tm_clones
     5: 00000000000010c0     0 FUNC    LOCAL  DEFAULT   16 register_tm_clones
     6: 0000000000001100     0 FUNC    LOCAL  DEFAULT   16 __do_global_dtors_aux
     7: 0000000000004010     1 OBJECT  LOCAL  DEFAULT   26 completed.0
     8: 0000000000003dc0     0 OBJECT  LOCAL  DEFAULT   22 __do_global_dtor[...]
     9: 0000000000001140     0 FUNC    LOCAL  DEFAULT   16 frame_dummy
    10: 0000000000003db8     0 OBJECT  LOCAL  DEFAULT   21 __frame_dummy_in[...]
    11: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS test.c
    12: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    13: 0000000000002110     0 OBJECT  LOCAL  DEFAULT   20 __FRAME_END__
    14: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS
    15: 0000000000003dc8     0 OBJECT  LOCAL  DEFAULT   23 _DYNAMIC
    16: 000000000000200c     0 NOTYPE  LOCAL  DEFAULT   19 __GNU_EH_FRAME_HDR
    17: 0000000000003fb8     0 OBJECT  LOCAL  DEFAULT   24 _GLOBAL_OFFSET_TABLE_
    18: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_mai[...]
    19: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterT[...]
    20: 0000000000004000     0 NOTYPE  WEAK   DEFAULT   25 data_start
    21: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@GLIBC_2.2.5
    22: 0000000000004010     0 NOTYPE  GLOBAL DEFAULT   25 _edata
    23: 0000000000001178     0 FUNC    GLOBAL HIDDEN    17 _fini
    24: 0000000000001149    26 FUNC    GLOBAL DEFAULT   16 hello
    25: 0000000000004000     0 NOTYPE  GLOBAL DEFAULT   25 __data_start
    26: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
    27: 0000000000004008     0 OBJECT  GLOBAL HIDDEN    25 __dso_handle
    28: 0000000000002000     4 OBJECT  GLOBAL DEFAULT   18 _IO_stdin_used
    29: 0000000000004018     0 NOTYPE  GLOBAL DEFAULT   26 _end
    30: 0000000000001060    38 FUNC    GLOBAL DEFAULT   16 _start
    31: 0000000000004010     0 NOTYPE  GLOBAL DEFAULT   26 __bss_start
    32: 0000000000001163    20 FUNC    GLOBAL DEFAULT   16 main
    33: 0000000000004010     0 OBJECT  GLOBAL HIDDEN    25 __TMC_END__
    34: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMC[...]
    35: 0000000000000000     0 FUNC    WEAK   DEFAULT  UND __cxa_finalize@G[...]
    36: 0000000000001000     0 FUNC    GLOBAL HIDDEN    12 _init
root@lima-ebpf-dev:~#
```


## <span class="section-num">5</span> 显示 .note 开头的 sections {#显示-dot-note-开头的-sections}

`readelf -n`: 显示core notes Section，即以 .note开头的 section，例如：

-   .note.gnu.property
-   .note.gnu.build-id：包含用于查找debug file的 build-id；
-   .note.ABI-tag

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
```


## <span class="section-num">6</span> 显示重定位 sections {#显示重定位-sections}

readefl -r/--relocs：显示重定位 section：

```shell
root@lima-ebpf-dev:~# readelf -r hello

Relocation section '.rela.dyn' at offset 0x550 contains 8 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000003db8  000000000008 R_X86_64_RELATIVE                    1140
000000003dc0  000000000008 R_X86_64_RELATIVE                    1100
000000004008  000000000008 R_X86_64_RELATIVE                    4008
000000003fd8  000100000006 R_X86_64_GLOB_DAT 0000000000000000 __libc_start_main@GLIBC_2.34 + 0
000000003fe0  000200000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_deregisterTM[...] + 0
000000003fe8  000400000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0
000000003ff0  000500000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_registerTMCl[...] + 0
000000003ff8  000600000006 R_X86_64_GLOB_DAT 0000000000000000 __cxa_finalize@GLIBC_2.2.5 + 0

Relocation section '.rela.plt' at offset 0x610 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000003fd0  000300000007 R_X86_64_JUMP_SLO 0000000000000000 puts@GLIBC_2.2.5 + 0
root@lima-ebpf-dev:~#
```


## <span class="section-num">7</span> 显示 gnu version {#显示-gnu-version}

readelf -V/--version-info

-   .gnu.version
-   .gnu.version_r

<!--listend-->

```shell
root@lima-ebpf-dev:~# readelf -V hello

Version symbols section '.gnu.version' contains 7 entries:
 Addr: 0x000000000000050e  Offset: 0x00050e  Link: 6 (.dynsym)
  000:   0 (*local*)       2 (GLIBC_2.34)    1 (*global*)      3 (GLIBC_2.2.5)
  004:   1 (*global*)      1 (*global*)      3 (GLIBC_2.2.5)

Version needs section '.gnu.version_r' contains 1 entry:
 Addr: 0x0000000000000520  Offset: 0x000520  Link: 7 (.dynstr)
  000000: Version: 1  File: libc.so.6  Cnt: 2
  0x0010:   Name: GLIBC_2.2.5  Flags: none  Version: 3
  0x0020:   Name: GLIBC_2.34  Flags: none  Version: 2
```


## <span class="section-num">8</span> 打印指定 Sections 内容 {#打印指定-sections-内容}

-   `readelf -x/--hex-dump=<number|name>`: 使用 16 进制打印指定 Sections 的内容。
-   ~readelf -p/--string-dump=&lt;number|name&gt;~：使用文本打印指定 Sections的内容；
-   可以使用readelf -S来显示所有sections number和 name，同时有 S 标志的表示是字符串 Section：

<!--listend-->

```shell
root@lima-ebpf-dev:~# readelf -S hello |grep comm
  [27] .comment          PROGBITS         0000000000000000  00003010
root@lima-ebpf-dev:~# readelf -p .comment hello

String dump of section '.comment':
  [     0]  GCC: (Ubuntu 11.3.0-1ubuntu1~22.04.1) 11.3.0
```


## <span class="section-num">9</span> 显示 DWARF 内容 {#显示-dwarf-内容}

readelf -w 打印二进制的 DWARF 内容，即各种以.debug_XX开头的 Section 内容：

-   .eh_frame: 在C++ exception resolution场景使用；
-   会查找 debuginfo 文件来获得 DWARF 内容；

<!--listend-->

```shell
root@lima-ebpf-dev:~# readelf -w hello
Contents of the .eh_frame section:


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
...
Contents of the .debug_line_str section:

  0x00000000 74657374 2e63002f 726f6f74 00       test.c./root.

root@lima-ebpf-dev:~#
```

readelf -wX 或则readelf --debug-dump=YY来打印对应DWARF debug section的内容。

-   X/=YY，例如 -wa 等效为 --debug-dump abbrev
-   -w --debug-dump[a/=abbrev, A/=addr, r/=aranges, c/=cu_index, L/=decodedline, f/=frames, F/=frames-interp,
    g/=gdb_index, i/=info, o/=loc, m/=macro, p/=pubnames, t/=pubtypes, R/=Ranges, l/=rawline, s/=str,
    O/=str-offsets, u/=trace_abbrev, T/=trace_aranges, U/=trace_info]

    Display the contents of DWARF debug sections

例子：

```shell
root@lima-ebpf-dev:~# readelf --debug-dump=abbrev hello |head -6
Contents of the .debug_abbrev section (loaded from hello):

  Number TAG (0x0)
   1      DW_TAG_base_type    [no children]
    DW_AT_byte_size    DW_FORM_data1
    DW_AT_encoding     DW_FORM_data1
root@lima-ebpf-dev:~# readelf -wa hello |head -6
Contents of the .debug_abbrev section (loaded from hello):

  Number TAG (0x0)
   1      DW_TAG_base_type    [no children]
    DW_AT_byte_size    DW_FORM_data1
    DW_AT_encoding     DW_FORM_data1
root@lima-ebpf-dev:~#
```

如果二进制被 strip ，则本身不再包含调试符号表，这时 readelf会根据.gnu_debuglink Sections中的 debug 文件名（需要单独添加该 Section ），或根据 .note.gnu.build-id在 /usr/lib/debug 下查找单独的 debug file。
