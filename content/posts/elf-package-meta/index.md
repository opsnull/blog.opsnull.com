---
title: "add package meta to elf file"
author: ["opsnull"]
date: 2023-08-06T00:00:00+08:00
lastmod: 2024-02-19T21:07:29+08:00
tags: ["linux", "elf", "tools"]
categories: ["elf", "tools"]
draft: false
series: ["elf-debug"]
series_order: 5
---

介绍向 elf 二进制文件中添加自定义 package meta 信息的方法。

<!--more-->


## <span class="section-num">1</span> 二进制文件添加自定义 note {#二进制文件添加自定义-note}

ELF 每个 note section 的开始部分应该包含一个 header，其中包含名字，描述以及类型的长度。然后，接着这个 header
的就是名字和描述的内容。同时，为了内存对齐，每个部分可能需要用额外的 null 字节进行填充。

-   不对齐在读取时报错：llvm-readelf: error: 'exec.ok': SHT_STRTAB string table section [index 3] is non-null
    terminated

为了实现 null 字节填充对齐，这里使用一个 python 脚本来实现：

```python
root@lima-ebpf-dev: # cat mynote.py
import struct

name = b"MyNote"
desc = b"This is some metadata"
types = 1

namesz = len(name) + 1  # Include trailing null byte
descsz = len(desc) + 1  # Include trailing null byte

# Note header
header = struct.pack("III", namesz, descsz, types)
# Name field (null-terminated, and padded to a multiple of 4)
name_field = name + b'\0' + b'\0' * ((4 - namesz % 4) % 4)
# Desc field (null-terminated, and padded to a multiple of 4)
desc_field = desc + b'\0' + b'\0' * ((4 - descsz % 4) % 4)

with open("note.bin", "wb") as f:
    f.write(header)
    f.write(name_field)
    f.write(desc_field)
root@lima-ebpf-dev:#
```

```shell
root@lima-ebpf-dev:# python3 mynote.py #创建自定义签名
root@lima-ebpf-dev:# cat note.bin
MyNoteThis is some metadataroot@lima-ebpf-dev:#

root@lima-ebpf-dev:# objcopy --add-section .note.mynote2=note.bin ./exec  # 将签名添加到 binary ELF 文件的

root@lima-ebpf-dev:# readelf -n ./exec  # 读取 ELF 文件中添加的签名

Displaying notes found in: .note.go.buildid
  Owner                Data size        Description
  Go                   0x00000053       GO BUILDID
   description data: 42 73 46 43 72 7a 5a 6b 6b 6c 6e 78 64 79 65 58 42 51 37 46 2f 39 6c 74 79 48 46 2d 4f 70 67 46 41 77 6b 78 34 4c 2d 6b 66 2f 38 56 4f 56 4c 44 44 77 33 4e 6c 57 74 46 45 73 66 77 65 35 2f 67 32 56 63 50 44 58 53 6a 39 33 36 75 49 35 6f 6f 31 74 55

Displaying notes found in: .note.mynote
  Owner                Data size        Description
  MyNote               0x00000016       NT_VERSION (version)
   description data: 54 68 69 73 20 69 73 20 73 6f 6d 65 20 6d 65 74 61 64 61 74 61 00

Displaying notes found in: .note.mynote2  # 添加的自定义签名
  Owner                Data size        Description
  MyNote               0x00000016       NT_VERSION (version)
   description data:

54 68 69 73 20 69 73 20 73 6f 6d 65 20 6d 65 74 61 64 61 74 61 00
root@lima-ebpf-dev:#

root@lima-ebpf-dev:# echo "5468697320697320736f6d65206d6574616461746100" | xxd -r -p # 解码，验证一致。
This is some metadataroot@lima-ebpf-dev:#
```

参考：

1.
2.  [how-to-replace-a-section-of-an-elf-file-with-another-using-objcopy-or-libelf-su](https://stackoverflow.com/questions/38647589/how-to-replace-a-section-of-an-elf-file-with-another-using-objcopy-or-libelf-su)


## <span class="section-num">2</span> 链接时生成 package meta {#链接时生成-package-meta}

已有机制：.note.gnu.build-id，可以根据 build-id 来使用 dnf repoquery --wathprovides
debuginfo(build-id_= XXX 来反查对应的 package。

缺点：

1.  build-id 的信息对用于来说太少；
2.  需要使用 rpm database 来反查 package；

新的机制： `.note.package`

-   在编译构建时注入该信息；
-   内容为 json 字符串，参考 [COREDUMP_PACKAGE_METADATA](https://systemd.io/COREDUMP_PACKAGE_METADATA/)；

<!--listend-->

```shell
$ objdump -s -j .note.package build/libhello.so

build/libhello.so:     file format elf64-x86-64

Contents of section .note.package:
 02ec 04000000 63000000 7e1afeca 46444f00  ....c...~...FDO.
 02fc 7b227479 7065223a 2272706d 222c226e  {"type":"rpm","n
 030c 616d6522 3a226865 6c6c6f22 2c227665  ame":"hello","ve
 031c 7273696f 6e223a22 302d312e 66633335  rsion":"0-1.fc35
 032c 2e783836 5f363422 2c226f73 43706522  .x86_64","osCpe"
 033c 3a226370 653a2f6f 3a666564 6f726170  :"cpe:/o:fedorap
 034c 726f6a65 63743a66 65646f72 613a3333  roject:fedora:33
 035c 227d0000

$ readelf --notes build/hello | grep "description data" | sed -e "s/\s*description data: //g" -e "s/ //g" | xxd -p -r | jq
readelf: build/hello: Warning: Gap in build notes detected from 0x1091 to 0x10de
readelf: build/hello: Warning: Gap in build notes detected from 0x1091 to 0x10af
readelf: build/hello: Warning: Gap in build notes detected from 0x1091 to 0x119f
{
  "type": "rpm",
  "name": "hello",
  "version": "0-1.fc35.x86_64",
  "osCpe": "cpe:/o:fedoraproject:fedora:33"
}

$ coredumpctl info
           PID: 44522 (fsverity)
...
       Package: fsverity-utils/1.3-1
      build-id: ac89bf7175b04d7eec7f6544a923f45be111f0be
       Message: Process 44522 (fsverity) of user 1000 dumped core.

                Found module /home/bluca/git/fsverity-utils/libfsverity.so.0 with build-id: fa40fdfb79aea84167c98ca8a89add9ac4f51069
                Metadata for module /home/bluca/git/fsverity-utils/libfsverity.so.0 owned by FDO found: {
                	"packageType" : "deb",
                	"package" : "fsverity-utils",
                	"packageVersion" : "1.3-1"
                }

                Found module linux-vdso.so.1 with build-id: aba08e06103f725e26f1d7c178fb6b76a564a35d
                Found module libpthread.so.0 with build-id: e91114987a0147bd050addbd591eb8994b29f4b3
                Found module libdl.so.2 with build-id: d3583c742dd47aaa860c5ae0c0c5bdbcd2d54f61
                Found module ld-linux-x86-64.so.2 with build-id: f25dfd7b95be4ba386fd71080accae8c0732b711
                Found module libcrypto.so.1.1 with build-id: 749142d5ee728a76e7cdc61fd79d2311a77405a2
                Found module libc.so.6 with build-id: 18b9a9a8c523e5cfe5b5d946d605d09242f09798
                Found module fsverity with build-id: ac89bf7175b04d7eec7f6544a923f45be111f0be
                Metadata for module fsverity owned by FDO found: {
                	"packageType" : "deb",
                	"package" : "fsverity-utils",
                	"packageVersion" : "1.3-1"
                }

                Stack trace of thread 44522:
                #0  0x00007fe7c8af26f4 __GI___nanosleep (libc.so.6 + 0xc66f4)
                #1  0x00007fe7c8af262a __sleep (libc.so.6 + 0xc662a)
                #2  0x00005608481407dd main (fsverity + 0x27dd)
                #3  0x00007fe7c8a5009b __libc_start_main (libc.so.6 + 0x2409b)
                #4  0x000056084814094a _start (fsverity + 0x294a)
```

JSON payload:

```json
{
     "type":"rpm",          # this provides a namespace for the package+package-version fields
     "os":"fedora",
     "osVersion":"33",
     "name":"coreutils",
     "version":"4711.0815.fc13",
     "architecture":"arm32",
     "osCpe": "cpe:/o:fedoraproject:fedora:33",          # A CPE name for the operating system, `CPE_NAME` from os-release is a good default
     "debugInfoUrl": "https://debuginfod.fedoraproject.org/"
}
```

实现方案: [ld 链接时传递 `--package-metadata` 参数](https://github.com/systemd/package-notes/blob/main/rpm/redhat-package-notes.in)， make -j4 V=1 LDFLAGS="-static -all-static -specs=/build/package-metadata.spec"，其中
package-metadata.spec 内容如下：

```shell
*link:
+ --package-metadata={\"type\":\"rpm\",\"name\":\"%:getenv(RPM_PACKAGE_NAME \",\"version\":\"%:getenv(RPM_PACKAGE_VERSION -%:getenv(RPM_PACKAGE_RELEASE \",\"architecture\":\"%:getenv(RPM_ARCH \",\"osCpe\":\"@OSCPE@\"}))))
```

A reference implementations of `a build-time tool is provided and can be used to generate a linker script`,
which can then be used at build time via `LDFLAGS="-Wl,-T,/path/to/generated/script"` to include the note in the
binary.

-   C 链接时使用: LDFLAGS="-Wl,-T,/path/to/generated/script"
-   GO 链接时使用: CGO_ENABLED=1 GOOS=linux GOARCH=amd64 GO111MODULE=on go build -o exec -ldflags "-linkmode
    external -extld gcc -extldflags '-static -v -Xlinker -T./module_info.ld'"

Generator:

-   <https://github.com/microsoft/CBL-Mariner/blob/dev/SPECS/mariner-rpm-macros/generate-package-note.py>
-   <https://github.com/systemd/package-notes/issues/13>

不太建议: 因为需要将生成的 c 文件和源代码一块编译链接.

1.  会生成 4 个文件  module_info.ld, auto_module_info.h, module_info.c 和 .note.pakage.bin
2.  module_info.c 包含 elf .note.package section 的具体内容, module_info.ld 是 ld 链接脚本;
3.  需要将 module_info.c 的内容和二进制一块编译链接;

generate-package-note.py 生成 ld 链接脚本：

```shell
./generate-package-note.py --name "exec" --type "bb" --version "1.2.3.4" \
			   --moduleVersion "1.2.3.4-beta" --os "mariner" \
			   --osVersion "1.0" --maintainer "xx" --copyright "xxx" \
			   --outdir "./" --repo "pkgname-git-repo-name" --hash "xxxx"  \
			   --branch "1.0" --stamp "Mix"

# ls -lat |head
-rw-r--r--  1 root root     277 Jul 17 04:22 auto_module_info.h
-rw-r--r--  1 root root    2082 Jul 17 04:22 module_info.c
drwxr-xr-x 28 root root     896 Jul 17 04:22 .
-rw-r--r--  1 root root     738 Jul 17 04:22 module_info.ld
-rw-r--r--  1 root root     260 Jul 17 04:22 .note.package.bin
drwxr-xr-x 20 root root     640 Jul 17 04:15 ..
-rw-r--r--  1 root root  315628 Jul 17 03:55 exec
-rwxr-xr-x  1 root root   13931 Jul 17 03:50 generate-package-note.py
-rw-r--r--  1 root root 1977702 Jul 17 03:41 exec.skel.h

root@lima-ebpf-dev:# cat module_info.c
const unsigned char __attribute__((aligned(4), section(".note.package"))) __attribute__((used)) module_info_note_package[] = {
                0x04,  0x00,  0x00,  0x00,
                0xf4,  0x00,  0x00,  0x00,
                0x7e,  0x1a,  0xfe,  0xca,
                0x46,  0x44,  0x4f,  0x00,
                0x7b,  0x0a,  0x20,  0x22,
                0x62,  0x72,  0x61,  0x6e,
                0x63,  0x68,  0x22,  0x3a,
                0x20,  0x22,  0x31,  0x2e,
                0x30,  0x22,  0x2c,  0x0a,

root@lima-ebpf-dev:# cat auto_module_info.h
#ifndef _AUTO_MODULE_INFO_H_
#define _AUTO_MODULE_INFO_H_

#define MODULE_VERSION      "1.2.3.4-beta"
#define PACKAGE_VERSION     "1.2.3.4"
#define PACKAGE_NAME        "exec"
#define TARGET_OS           "mariner"
#define TARGET_OS_VERSION   "1.0"

#endif //_AUTO_MODULE_INFO_H_

root@lima-ebpf-dev:# cat module_info.ld
/*

    This file is automatically generated by generate-package-note.py tool.
    Do not modify this file, your changes will be lost!

*/

/*
./generate-package-note.py --name exec --type bb --version 1.2.3.4 --moduleVersion 1.2.3.4-beta --os mariner --osVersion 1.0 --maintainer xx --copyright xxx --outdir ./ --repo pkgname-git-repo-name --hash xxxx --branch 1.0 --stamp Mix
*/

/*
{
 "branch": "1.0",
 "copyright": "xxx",
 "hash": "xxxx",
 "maintainer": "xx",
 "moduleVersion": "1.2.3.4-beta",
 "name": "exec",
 "os": "mariner",
 "osVersion": "1.0",
 "repo": "pkgname-git-repo-name",
 "type": "bb",
 "version": "1.2.3.4"
}
*/

SECTIONS
{
        .note.package : ALIGN(4)
        {

                KEEP (*(.note.package))
        }
}
INSERT AFTER .note.gnu.build-id;

root@lima-ebpf-dev:# cat .note.package.bin
{
 "branch": "1.0",
 "copyright": "xxx",
 "hash": "xxxx",
 "maintainer": "xx",
 "moduleVersion": "1.2.3.4-beta",
 "name": "exec",
 "os": "mariner",
 "osVersion": "1.0",
 "repo": "pkgname-git-repo-name",
 "type": "bb",
 "version": "1.2.3.4"
}
```

编译链接命令:

```shell
CGO_ENABLED=1 GOOS=linux GOARCH=amd64 GO111MODULE=on go build -o exec -ldflags "-linkmode external -extld gcc
-extldflags '-static -v -Xlinker -T./module_info.ld'"
```

缺点: --package-metadata [需要较高的 binutils 版本](https://github.com/systemd/package-notes/):

-   binutils (&gt;= 2.39): 提供 ld
-   mold (&gt;= 1.3.0)
-   lld (&gt;= 15.0.0)

ubuntu 22.04 和 alios7 的 binutils 版本低, 都不支持 --package-metadata:

```shell
#ubuntu 22.04
root@lima-ebpf-dev:/# ld --version
GNU ld (GNU Binutils for Ubuntu) 2.38
Copyright (C) 2022 Free Software Foundation, Inc.
This program is free software; you may redistribute it under the terms of
the GNU General Public License version 3 or (at your option) a later version.
This program has absolutely no warranty.

# alios7
# ld --version
GNU ld version 2.27-41.base.2.alios7.1
Copyright (C) 2016 Free Software Foundation, Inc.
This program is free software; you may redistribute it under the terms of
the GNU General Public License version 3 or (at your option) a later version.
This program has absolutely no warranty.
```

参考：

1.  [LWN: Adding package information to ELF objects](https://lwn.net/Articles/874642/)
2.  [fedora：Package_information_on_ELF_objects](https://www.fedoraproject.org/wiki/Changes/Package_information_on_ELF_objects)
