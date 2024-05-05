---
title: "Rust ESP32 工具链安装和测试"
author: ["张俊(zj@opsnull.com)"]
date: 2024-02-19T00:00:00+08:00
lastmod: 2024-05-05T17:56:07+08:00
tags: ["rust", "esp32"]
categories: ["rust", "esp32"]
draft: false
series: ["rust-esp32"]
series_order: 1
---

## <span class="section-num">1</span> esp-idf 安装 {#esp-idf-安装}

[安装 ESP-IDF](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/linux-macos-setup.html#for-macos-users) 到 `~/.espressif/` 目录:

```shell
$ brew install cmake ninja dfu-util
$ brew install ccache # 可选, 加快构建速度
$ echo 'PATH=/opt/homebrew/opt/ccache/libexec:$PATH' >>~/.bashrc

# 确保系统是 python 3 版本且没有激活 venv，否则后续执行 install.sh 脚本会失败。
# 执行完 install.sh 脚本后，可以继续激活 venv。
$ which python3
/Users/alizj/.venv/bin//python3
$ source /Users/alizj/.venv/bin/activate && deactive
# 从 PATH 中临时删除 /Users/alizj/.venv/bin 目录。
$ export DIR_TO_REMOVE=/Users/alizj/.venv/bin
$ export PATH=$(echo $PATH | sed -e "s;:$DIR_TO_REMOVE;;" -e "s;$DIR_TO_REMOVE:;;" -e "s;$DIR_TO_REMOVE;;")

$ mkdir -p ~/esp
$ cd ~/esp
$ git clone --recursive https://github.com/espressif/esp-idf.git
$ cd ~/esp/esp-idf

# python3 不支持 SOCKS5 代理，否则执行下面的 install.sh 脚本会出错。
$ enable_http_proxy
$ ./install.sh esp32s3  # esp32,esp32s2 等目标芯片, all 表示所有.
Detecting the Python interpreter
Checking "python3" ...
Python 3.12.3
"python3" has been detected
Checking Python compatibility
Installing ESP-IDF tools
Selected targets are: esp32s3
Current system platform: macos-arm64
Installing tools: xtensa-esp-elf-gdb, xtensa-esp-elf, riscv32-esp-elf, esp32ulp-elf, openocd-esp32, esp-rom-elfs
Skipping xtensa-esp-elf-gdb@14.2_20240403 (already installed)
Skipping xtensa-esp-elf@esp-13.2.0_20240305 (already installed)
Skipping riscv32-esp-elf@esp-13.2.0_20240305 (already installed)
Skipping esp32ulp-elf@2.38_20240113 (already installed)
Skipping openocd-esp32@v0.12.0-esp32-20240318 (already installed)
Skipping esp-rom-elfs@20240305 (already installed)
Installing Python environment and packages
Creating a new Python environment in /Users/alizj/.espressif/python_env/idf5.4_py3.12_env
Downloading https://dl.espressif.com/dl/esp-idf/espidf.constraints.v5.4.txt
Destination: /Users/alizj/.espressif/espidf.constraints.v5.4.txt.tmp
Done
Installing Python packages
 Constraint file: /Users/alizj/.espressif/espidf.constraints.v5.4.txt
 Requirement files:
  - /Users/alizj/esp/esp-idf/tools/requirements/requirements.core.txt
Looking in indexes: https://pypi.org/simple, https://dl.espressif.com/pypi
Ignoring importlib_metadata: markers 'python_version < "3.8"' don't match your environment
Collecting setuptools (from -r /Users/alizj/esp/esp-idf/tools/requirements/requirements.core.txt (line 3))
。。。
All done! You can now run:

  . ./export.sh
```

安装的内容位于 `~/.espressif` 目录下：

openocd-esp32/
: esp32 fork 的 openocd 版本；

riscv32-esp-elf/
: riscv32 交叉编译工具链；

xtensa-esp-elf/
: xtensa 交叉编译工具链；

xtensa-esp-elf-gdb/
: xtensa gdb 调试工具；

<!--listend-->

```shell
zj@a:~/esp/esp-idf$ ls -l ~/.espressif/
total 8.0K
drwxr-xr-x 8 alizj  256  5  5 11:31 dist/
-rw-r--r-- 1 alizj 2.8K  5  5 11:36 espidf.constraints.v5.4.txt
-rw-r--r-- 1 alizj  291  5  5 11:20 idf-env.json
drwxr-xr-x 3 alizj   96  5  5 11:36 python_env/
drwxr-xr-x 8 alizj  256  5  5 11:31 tools/
zj@a:~/esp/esp-idf$ ls -l ~/.espressif/dist/
total 297M
-rw-r--r-- 1 alizj 3.2M  5  5 11:31 esp-rom-elfs-20240305.tar.gz
-rw-r--r-- 1 alizj  16M  5  5 11:31 esp32ulp-elf-2.38_20240113-macos-arm64.tar.gz
-rw-r--r-- 1 alizj 2.3M  5  5 11:31 openocd-esp32-macos-arm64-0.12.0-esp32-20240318.tar.gz
-rw-r--r-- 1 alizj 131M  5  5 11:28 riscv32-esp-elf-13.2.0_20240305-aarch64-apple-darwin.tar.xz
-rw-r--r-- 1 alizj  96M  5  5 11:21 xtensa-esp-elf-13.2.0_20240305-aarch64-apple-darwin.tar.xz
-rw-r--r-- 1 alizj  36M  5  5 11:20 xtensa-esp-elf-gdb-14.2_20240403-aarch64-apple-darwin21.1.tar.gz
zj@a:~/esp/esp-idf$ ls -l ~/.espressif/tools/
total 0
drwxr-xr-x 3 alizj 96  5  5 11:31 esp-rom-elfs/
drwxr-xr-x 3 alizj 96  5  5 11:31 esp32ulp-elf/
drwxr-xr-x 3 alizj 96  5  5 11:31 openocd-esp32/
drwxr-xr-x 3 alizj 96  5  5 11:28 riscv32-esp-elf/
drwxr-xr-x 3 alizj 96  5  5 11:21 xtensa-esp-elf/
drwxr-xr-x 3 alizj 96  5  5 11:20 xtensa-esp-elf-gdb/
```

后续每次使用 esp-idf 前需要 `source ~/esp/esp-idf/export.sh` ：

```shell
# export.sh 脚本不能移动位置，必须 source 使用。
zj@a:~/esp/esp-idf$ source ~/esp/esp-idf/export.sh
Detecting the Python interpreter
Checking "python3" ...
Python 3.12.3
"python3" has been detected
Checking Python compatibility
Checking other ESP-IDF version.
Adding ESP-IDF tools to PATH...
Checking if Python packages are up to date...
Constraint file: /Users/alizj/.espressif/espidf.constraints.v5.4.txt
Requirement files:
 - /Users/alizj/esp/esp-idf/tools/requirements/requirements.core.txt
Python being checked: /Users/alizj/.espressif/python_env/idf5.4_py3.12_env/bin/python
Python requirements are satisfied.
Added the following directories to PATH:
  /Users/alizj/esp/esp-idf/components/espcoredump
  /Users/alizj/esp/esp-idf/components/partition_table
  /Users/alizj/esp/esp-idf/components/app_update
  /Users/alizj/.espressif/tools/xtensa-esp-elf-gdb/14.2_20240403/xtensa-esp-elf-gdb/bin
  /Users/alizj/.espressif/tools/xtensa-esp-elf/esp-13.2.0_20240305/xtensa-esp-elf/bin
  /Users/alizj/.espressif/tools/riscv32-esp-elf/esp-13.2.0_20240305/riscv32-esp-elf/bin
  /Users/alizj/.espressif/tools/esp32ulp-elf/2.38_20240113/esp32ulp-elf/bin
  /Users/alizj/.espressif/tools/openocd-esp32/v0.12.0-esp32-20240318/openocd-esp32/bin
  /Users/alizj/.espressif/tools/xtensa-esp-elf-gdb/14.2_20240403/xtensa-esp-elf-gdb/bin
  /Users/alizj/.espressif/tools/xtensa-esp-elf/esp-13.2.0_20240305/xtensa-esp-elf/bin
  /Users/alizj/.espressif/tools/riscv32-esp-elf/esp-13.2.0_20240305/riscv32-esp-elf/bin
  /Users/alizj/.espressif/tools/esp32ulp-elf/2.38_20240113/esp32ulp-elf/bin
  /Users/alizj/.espressif/tools/openocd-esp32/v0.12.0-esp32-20240318/openocd-esp32/bin
Done! You can now compile ESP-IDF projects.
Go to the project directory and run:

  idf.py build

zj@a:~/esp/esp-idf$ which xtensa-esp32s3-elf-gcc # 来源于 xtensa-esp-elf
/Users/alizj/.espressif/tools/xtensa-esp-elf/esp-13.2.0_20240305/xtensa-esp-elf/bin/xtensa-esp32s3-elf-gcc

zj@a:~/esp/esp-idf$ which xtensa-esp32s3-elf-gdb # 来源于 xtensa-esp-elf-gdb
/Users/alizj/.espressif/tools/xtensa-esp-elf-gdb/14.2_20240403/xtensa-esp-elf-gdb/bin/xtensa-esp32s3-elf-gdb

zj@a:~/esp/esp-idf$ ls ~/.espressif/
dist/                        espidf.constraints.v5.4.txt  idf-env.json                 python_env/                  tools/
zj@a:~/esp/esp-idf$ ls ~/.espressif/tools/
esp-rom-elfs/  esp32ulp-elf/  openocd-esp32/  riscv32-esp-elf/  xtensa-esp-elf/  xtensa-esp-elf-gdb/
zj@a:~/esp/esp-idf$ xtensa-esp32s3-elf-
xtensa-esp32s3-elf-addr2line   xtensa-esp32s3-elf-cc          xtensa-esp32s3-elf-gcc-13.2.0  xtensa-esp32s3-elf-gcov-dump   xtensa-esp32s3-elf-ld.bfd      xtensa-esp32s3-elf-ranlib
xtensa-esp32s3-elf-ar          xtensa-esp32s3-elf-cpp         xtensa-esp32s3-elf-gcc-ar      xtensa-esp32s3-elf-gcov-tool   xtensa-esp32s3-elf-lto-dump    xtensa-esp32s3-elf-readelf
xtensa-esp32s3-elf-as          xtensa-esp32s3-elf-elfedit     xtensa-esp32s3-elf-gcc-nm      xtensa-esp32s3-elf-gdb         xtensa-esp32s3-elf-nm          xtensa-esp32s3-elf-size
xtensa-esp32s3-elf-c++         xtensa-esp32s3-elf-g++         xtensa-esp32s3-elf-gcc-ranlib  xtensa-esp32s3-elf-gprof       xtensa-esp32s3-elf-objcopy     xtensa-esp32s3-elf-strings
xtensa-esp32s3-elf-c++filt     xtensa-esp32s3-elf-gcc         xtensa-esp32s3-elf-gcov        xtensa-esp32s3-elf-ld          xtensa-esp32s3-elf-objdump     xtensa-esp32s3-elf-strip
```


## <span class="section-num">2</span> esp-idf 开发 {#esp-idf-开发}

除了使用 c/c++ 原生方式使用 esp-idf 外，在进行 Rust 和 c/++ 混合编程时（如 cargo generate
esp-rs/esp-idf-template cmake) 时也使用 esp-idf 的开发、构建和烧写工具 idf.py:

-   <https://github.com/esp-rs/esp-idf-template/blob/master/README-cmake.md>

ESP-IDF (Espressif IoT Development Framework)  [开发者文档](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/index.html)：

-   idf.py menuconfig 生成 sdkconfig，具体的配置参数列表：[kconfig.html](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-reference/kconfig.html)
-   JTAG Debugging: <https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-guides/jtag-debugging/index.html>

hello-world 示例：

```shell
zj@a:~$ source ~/esp/esp-idf/export.sh  # 每次使用 esp-dif 前必须 source 安装路径中的 export.sh

zj@a:~/esp/esp-idf$ cd examples/get-started/hello_world/
zj@a:~/esp/esp-idf/examples/get-started/hello_world$ ls
CMakeLists.txt  README.md  main/  pytest_hello_world.py  sdkconfig.ci

# 清理项目已经存在的 build 和 configure，生成新的 sdkconfig 文件
zj@a:~/esp/esp-idf/examples/get-started/hello_world$ idf.py set-target esp32s3

# 配置项目，结果保存到 sdkconfig 文件
zj@a:~/esp/esp-idf/examples/get-started/hello_world$ idf.py menuconfig
Executing action: menuconfig
Running ninja in directory /Users/alizj/esp/esp-idf/examples/get-started/hello_world/build
Executing "ninja menuconfig"...
[0/1] cd /Users/alizj/esp/esp-idf/examples/get-started/hello_world/build && /Users/alizj/.espressif/pytho...DF_INIT_VERSION=5.4.0 --output config /Users/alizj/esp/esp-idf/examples/get-started/hello_world/sdkconfi
TERM environment variable is set to "xterm-256color"
Loaded configuration '/Users/alizj/esp/esp-idf/examples/get-started/hello_world/sdkconfig'
No changes to save (for '/Users/alizj/esp/esp-idf/examples/get-started/hello_world/sdkconfig')

# 构建
zj@a:~/esp/esp-idf/examples/get-started/hello_world$ idf.py build
。。。
[998/1000] Generating binary image from built executable
esptool.py v4.8.dev3
Creating esp32s3 image...
Merged 3 ELF sections
Successfully created esp32s3 image.
Generated /Users/alizj/esp/esp-idf/examples/get-started/hello_world/build/hello_world.bin
[999/1000] cd /Users/alizj/esp/esp-idf/examples/get-started/hello_world/build/esp-idf/esptool_py && /User...table/partition-table.bin /Users/alizj/esp/esp-idf/examples/get-started/hello_world/build/hello_world.bi
hello_world.bin binary size 0x35a00 bytes. Smallest app partition is 0x100000 bytes. 0xca600 bytes (79%) free.

Project build complete. To flash, run:
 idf.py flash
or
 idf.py -p PORT flash
or
 python -m esptool --chip esp32s3 -b 460800 --before default_reset --after hard_reset write_flash --flash_mode dio --flash_size 2MB --flash_freq 80m 0x0 build/bootloader/bootloader.bin 0x8000 build/partition_table/partition-table.bin 0x10000 build/hello_world.bin
or from the "/Users/alizj/esp/esp-idf/examples/get-started/hello_world/build" directory
 python -m esptool --chip esp32s3 -b 460800 --before default_reset --after hard_reset write_flash "@flash_args"
```

烧写到设备 Flash：

-   USB 串口：linux: /dev/ttyUSB0, macOS: /dev/cu.\*
-   烧写了三个文件：
    1.  0x0 build/bootloader/bootloader.bin
    2.  0x8000 build/partition_table/partition-table.bin
    3.  0x10000 build/hello_world.bin

<!--listend-->

```shell

$ idf.py -p PORT flash # flash 命令自动 build 和 flash：

# 监控串口输出
$ idf.py -p <PORT> monitor
Running idf_monitor in directory [...]/esp/hello_world/build
Executing "python [...]/esp-idf/tools/idf_monitor.py -b 115200 [...]/esp/hello_world/build/hello_world.elf"...
--- idf_monitor on <PORT> 115200 ---
--- Quit: Ctrl+] | Menu: Ctrl+T | Help: Ctrl+T followed by Ctrl+H ---
ets Jun  8 2016 00:22:57

rst:0x1 (POWERON_RESET),boot:0x13 (SPI_FAST_FLASH_BOOT)
ets Jun  8 2016 00:22:57
...

# 一条命令完成 build/flash/monitor
idf.py -p PORT flash monitor
```


## <span class="section-num">3</span> esp-rs 安装 {#esp-rs-安装}

参考：<https://esp-rs.github.io/book/introduction.https>

[BROKEN LINK: html://github.com/esp-rs/esp-idf-template?tab=readme-ov-file#prerequisites]:

-   不需要手动安装 esp-idf，后续构建 esp-rs 项目是 build.rs 会自动下载 esp-idf。

<!--listend-->

```shell
# 提供 cargo generate 子命令
cargo install cargo-generate

# A tool that forwards linker arguments to the actual linker that is also given as an argument to
# ldproxy
cargo install ldproxy

# A tool that simplifies installing and maintaining the components required to develop Rust
# applications for the Xtensa and RISC-V architectures.
cargo install espup

brew install libuv  # espflash 依赖
cargo install espflash
cargo install cargo-espflash # Optional，将 espflash 作为 cargo 的子命令来使用
```

espup 可以同时安装和维护 Xtensa and RISC-V architectures 的工具链，包括 esp fork 的 rust，GCC 和
LLVM 等：

-   espup 是 rust 开发的工具，取代了 rust-build 项目。

<!--listend-->

```shell
# 清理旧版本
zj@a:~/esp$ espup uninstall
[info]: Uninstalling the Espressif Rust ecosystem
[info]: Uninstalling Xtensa LLVM
[info]: Uninstalling GCC
[info]: Uninstalling Xtensa Rust toolchain
[info]: Uninstallation successfully completed!
zj@a:~/esp$ rm -rf  ~/.rustup/toolchains/*

# espup 是 Rust 程序，不会执行 python 代码，所以支持 socks 代理。开启代理，加快下载
zj@a:~/esp$ enable_socks_proxy
# 安装 ESP32 的 Rust toolchain。
zj@a:~/esp$ espup install -l debug
。。。
[debug]: Creating export file
[info]: Installation successfully completed!

        To get started, you need to set up some environment variables by running: '. /Users/alizj/export-esp.sh'
        This step must be done every time you open a new terminal.
            See other methods for setting the environment in https://esp-rs.github.io/book/installation/riscv-and-xtensa.html#3-set-up-the-environment-variables

zj@a:~$ cat ~/export-esp.sh
# 修改 export-esp.sh 脚本，删除 python venv 路径，防止后续 cargo build 安装 esp-idf 报错。
export DIR_TO_REMOVE=/Users/alizj/.venv/bin
PATH=$(echo "$PATH" | sed -e "s;:$DIR_TO_REMOVE;;" -e "s;$DIR_TO_REMOVE:;;" -e "s;$DIR_TO_REMOVE;;")

# 解决 Mac M1 cargo build 报错的问题：https://github.com/rust-lang/cc-rs/issues/1005
CRATE_CC_NO_DEFAULTS=1

# cargo build 过程中按照 esp-idf python venv 时不支持 SOCKS 代理，故切换为 HTTP 代理。
enable_http_proxy

export LIBCLANG_PATH="/Users/alizj/.rustup/toolchains/esp/xtensa-esp32-elf-clang/esp-16.0.4-20231113/esp-clang/lib"
export PATH="/Users/alizj/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin:$PATH"
```

为了方便使用，将 ~/export-esp.sh 脚本 mv 到 ~/esp 目录，同时添加两个 alias：

```shell
zj@a:~/esp$ mv ~/export-esp.sh ~/esp/
zj@a:~/esp$ grep esp ~/.bashrc
alias export_idf='. $HOME/esp/esp-idf/export.sh'
alias export_esp='. $HOME/esp/export-esp.sh'
```

espup 安装的内容位于 `~/.espup` 目录下（rustc 编译器本身是依赖 LLVM 的）：

-   Espressif Rust fork with support for Espressif targets
-   nightly toolchain with support for RISC-V targets
-   LLVM fork with support for Xtensa targets
-   GCC toolchain that links the final binary

<!--listend-->

```shell
# 安装了两个 toolchain，分别是 nightly-x86_64-apple-darwin 和 esp
# nightly-x86_64-apple-darwin 是 rust 官方工具链，支持 riscv32xx 和 x86_64-apple-darwin 等 4 个 targets
# 其中 riscv32xx 是 esp32-c3 系列 RISC-V CPU 类型。
zj@a:~/esp$ rustup show
Default host: aarch64-apple-darwin
rustup home:  /Users/alizj/.rustup

installed toolchains
--------------------

stable-aarch64-apple-darwin (default) # 标准工具链
nightly-aarch64-apple-darwin
esp # 自定义 esp 工具链

active toolchain
----------------

stable-aarch64-apple-darwin (default)
rustc 1.78.0 (9b00956e5 2024-04-29)

# espup 安装的 esp 工具链, esp 为 channel 名称
zj@a:~/esp$ ls -l ~/.rustup/toolchains/esp/
total 0
drwxr-xr-x 12 alizj 384  5  5 12:12 bin/ # esp fork 的 rust 交叉编译工具链
drwxr-xr-x  3 alizj  96  5  5 12:12 etc/
drwxr-xr-x  5 alizj 160  5  5 12:13 lib/
drwxr-xr-x  3 alizj  96  5  5 12:12 libexec/
drwxr-xr-x  5 alizj 160  5  5 12:12 share/
drwxr-xr-x  3 alizj  96  5  5 12:11 xtensa-esp-elf/ # esp fork 的支持 xtensa CPU 的 gcc 等工具链
drwxr-xr-x  3 alizj  96  5  5 12:11 xtensa-esp32-elf-clang/ # esp fork 的 clang LLVM 工具链

# esp fork 的 rust 交叉编译工具链
zj@a:~/esp$ ls -l ~/.rustup/toolchains/esp/bin/
total 63M
-rwxr-xr-x 1 alizj  30M  5  5 12:12 cargo*
-rwxr-xr-x 1 alizj 1.2M  5  5 12:12 cargo-clippy*
-rwxr-xr-x 1 alizj 1.6M  5  5 12:12 cargo-fmt*
-rwxr-xr-x 1 alizj  11M  5  5 12:12 clippy-driver*
-rwxr-xr-x 1 alizj  980  5  5 12:12 rust-gdb*
-rwxr-xr-x 1 alizj 2.2K  5  5 12:12 rust-gdbgui*
-rwxr-xr-x 1 alizj 1.1K  5  5 12:12 rust-lldb*
-rwxr-xr-x 1 alizj 584K  5  5 12:12 rustc*
-rwxr-xr-x 1 alizj  12M  5  5 12:12 rustdoc*
-rwxr-xr-x 1 alizj 7.1M  5  5 12:12 rustfmt*

zj@a:~/docs$ ls -l ~/.rustup/toolchains/esp/bin/
total 59M
-rwxr-xr-x 1 zhangjun  28M  2  8 14:51 cargo*
-rwxr-xr-x 1 zhangjun 1.1M  2  8 14:51 cargo-clippy*
-rwxr-xr-x 1 zhangjun 1.6M  2  8 14:51 cargo-fmt*
-rwxr-xr-x 1 zhangjun  11M  2  8 14:51 clippy-driver*
lrwxr-xr-x 1 zhangjun   80  2  8 16:58 rust-analyzer -> /Users/zhangjun/.rustup/toolchains/nightly-x86_64-apple-darwin/bin/rust-analyzer*  # 链接到官方工具链的 rust-analyzer
-rwxr-xr-x 1 zhangjun  980  2  8 14:50 rust-gdb*
-rwxr-xr-x 1 zhangjun 2.2K  2  8 14:50 rust-gdbgui*
-rwxr-xr-x 1 zhangjun 1.1K  2  8 14:50 rust-lldb*
-rwxr-xr-x 1 zhangjun 664K  2  8 14:50 rustc*
-rwxr-xr-x 1 zhangjun  11M  2  8 14:50 rustdoc*
-rwxr-xr-x 1 zhangjun 6.9M  2  8 14:51 rustfmt*

# esp rustc 支持 x86_64/arm64/riscv64/xtensa-esp32s3-espidf/xtensa-esp32s3-none-elf target
# 后续可以在项目的 rust-toolchain.toml 和 .cargo/config.toml 中指定使用 esp channel 和对应的 target。
zj@a:~/docs$  ~/.rustup/toolchains/esp/bin/rustc --print target-list |grep -E 'xtensa|riscv'
riscv32gc-unknown-linux-gnu
riscv32gc-unknown-linux-musl
riscv32i-unknown-none-elf
riscv32im-unknown-none-elf
riscv32imac-esp-espidf
riscv32imac-unknown-none-elf
riscv32imac-unknown-xous-elf
riscv32imc-esp-espidf
riscv32imc-unknown-none-elf
riscv64-linux-android
riscv64gc-unknown-freebsd
riscv64gc-unknown-fuchsia
riscv64gc-unknown-hermit
riscv64gc-unknown-linux-gnu
riscv64gc-unknown-linux-musl
riscv64gc-unknown-netbsd
riscv64gc-unknown-none-elf
riscv64gc-unknown-openbsd
riscv64imac-unknown-none-elf
xtensa-esp32-espidf
xtensa-esp32-none-elf
xtensa-esp32s2-espidf
xtensa-esp32s2-none-elf
xtensa-esp32s3-espidf  # esp idf target，即 std 应用
xtensa-esp32s3-none-elf # none-elf 为 non_std 应用
xtensa-esp8266-none-elf

# 后续使用 cargo generate esp-rs/esp-idf-template cargo 来生成 std 应用，
# esp32 rust 项目通过 rust-toolchain.toml 和 .cargo/config.toml 来选择 channel 和 target
zj@a:~/docs$ cat ~/codes/esp32/esp-demo2/myesp/rust-toolchain.toml
[toolchain]
channel = "esp"  # ~/.rustup/toolchains/ 下的目录名称，这里使用 esp toolchain

zj@a:~/codes/esp32/esp-demo2/myesp$ cat .cargo/config.toml
[build]
target = "xtensa-esp32s3-espidf"  # 要构建的 target，这里使用链接 esp-idf 的 target

[target.xtensa-esp32s3-espidf]  # target 对应的配置
linker = "ldproxy" # 位于 ~/.cargo/bin/
# runner = "espflash --monitor" # Select this runner for espflash v1.x.x
runner = "espflash flash --monitor" # Select this runner for espflash v2.x.x
rustflags = [ "--cfg",  "espidf_time64"] # Extending time_t for ESP IDF 5: https://github.com/esp-rs/rust/issues/110

zj@a:~/esp$ ls -l  ~/.espup/
total 0
lrwxr-xr-x 1 alizj 92  5  5 12:11 esp-clang -> /Users/alizj/.rustup/toolchains/esp/xtensa-esp32-elf-clang/esp-16.0.4-20231113/esp-clang/lib/

# ~/esp/export-esp.sh 脚本将 LIBCLANG_PATH 环境便利指向 esp fork 的 LLVM 目录，这样后续 rustc 在编译时自动
# 链接 esp 的版本。（rustc 依赖 LLVM）。
zj@a:~/esp$ ls -l  /Users/alizj/.rustup/toolchains/esp/xtensa-esp32-elf-clang/esp-16.0.4-20231113/esp-clang/lib/
total 117M
drwxr-xr-x 3 alizj  96  5  5 12:11 clang/
-rw-r--r-- 1 alizj 50M 11 14 23:46 libLLVM.dylib
-rw-r--r-- 1 alizj 44M 11 14 23:46 libclang-cpp.dylib
-rw-r--r-- 1 alizj 24M 11 14 23:46 libclang.dylib

zj@a:~/esp$ cd ~/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/
zj@a:~/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin$ ls -l *esp32s3*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-addr2line*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-ar*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-as*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-c++*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-c++filt*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-cc*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-cpp*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-elfedit*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-g++*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-gcc*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-gcc-13.2.0*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-gcc-ar*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-gcc-nm*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-gcc-ranlib*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-gcov*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-gcov-dump*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-gcov-tool*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-gprof*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-ld*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-ld.bfd*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-lto-dump*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-nm*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-objcopy*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-objdump*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-ranlib*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-readelf*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-size*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-strings*
-rwxr-xr-x 1 alizj 361K  9 29  2023 xtensa-esp32s3-elf-strip*
zj@a:~/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin$

# export-esp.sh 将该 bin 目录添加 PATH 前面
zj@a:~$ cd esp/
zj@a:~/esp$  ls ~/.rustup/toolchains/esp/xtensa-esp-elf/esp-*/xtensa-esp-elf/xtensa-esp-elf/bin/
ar*  as*  ld*  ld.bfd*  nm*  objcopy*  objdump*  ranlib*  readelf*  strip*

# 这些工具是 crosstool-NG 的 esp 版本
zj@a:~/docs$  ~/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/xtensa-esp-elf/bin/ld --version
GNU ld (crosstool-NG esp-13.2.0_20230928) 2.41
Copyright (C) 2023 Free Software Foundation, Inc.
# 它们支持 elf32-xtensa target
zj@a:~/docs$  ~/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/xtensa-esp-elf/bin/ld --help |grep supp
                              Enable support of non-contiguous memory regions
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/xtensa-esp-elf/bin/ld: supported targets: elf32-xtensa-le elf32-xtensa-be elf32-little elf32-big srec symbolsrec verilog tekhex binary ihex plugin
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/xtensa-esp-elf/bin/ld: supported emulations: elf32xtensa
```

后续每次使用 rust esp 时都需要 `source ~/esp/export-esp.sh` 环境变量:

```shell
zj@a:~/docs$ cat ~/esp/export-esp.sh
export DIR_TO_REMOVE=/Users/alizj/.venv/bin
PATH=$(echo "$PATH" | sed -e "s;:$DIR_TO_REMOVE;;" -e "s;$DIR_TO_REMOVE:;;" -e "s;$DIR_TO_REMOVE;;")

# 解决 Mac M1 cargo build 报错的问题：https://github.com/rust-lang/cc-rs/issues/1005
export CRATE_CC_NO_DEFAULTS=1

# cargo build 过程中按照 esp-idf python venv 时不支持 SOCKS 代理，故切换为 HTTP 代理。
enable_http_proxy

export LIBCLANG_PATH="/Users/alizj/.rustup/toolchains/esp/xtensa-esp32-elf-clang/esp-16.0.4-20231113/esp-clang/lib"
export PATH="/Users/alizj/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin:$PATH"
```

更新工具链：

```shell
zj@a:~/esp$ enable_socks_proxy
zj@a:~/esp$ espup update
[info]: Updating the Espressif Rust ecosystem
[info]: Checking Rust installation
[info]: Installing GCC (xtensa-esp-elf)
[warn]: Previous installation of LLVM exists in: '/Users/alizj/.rustup/toolchains/esp/xtensa-esp32-elf-clang/esp-16.0.4-20231113'. Reusing this installation
[info]: Installing RISC-V Rust targets ('riscv32imc-unknown-none-elf', 'riscv32imac-unknown-none-elf' and 'riscv32imafc-unknown-none-elf') for 'nightly' toolchain
[warn]: Previous installation of GCC exists in: '/Users/alizj/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928'. Reusing this installation
[info]: Creating symlink between '/Users/alizj/.rustup/toolchains/esp/xtensa-esp32-elf-clang/esp-16.0.4-20231113/esp-clang/lib' and '/Users/alizj/.espup/esp-clang'
[warn]: Previous installation of Xtensa Rust 1.77.0.0 exists in: '/Users/alizj/.rustup/toolchains/esp'. Reusing this installation
[info]: Update successfully completed!

        To get started, you need to set up some environment variables by running: '. /Users/alizj/export-esp.sh'
        This step must be done every time you open a new terminal.
        See other methods for setting the environment in https://esp-rs.github.io/book/installation/riscv-and-xtensa.html#3-set-up-the-environment-variables
```

esp 是 espup 安装的自定义 toolchain，不能直接安装 rust-analyzer， 需要建一个软链接：

-   <https://www.reddit.com/r/rust/comments/13d2tls/rustanalyzer_and_toolchain_for_esp/>

<!--listend-->

```rust
zj@a:~$ rustup component add rust-analyzer
info: downloading component 'rust-analyzer'
info: installing component 'rust-analyzer'

zj@a:~/docs$ ls -l ~/.rustup/toolchains/stable-aarch64-apple-darwin/bin/
total 96M
-rwxr-xr-x 1 alizj  27M  5  5 12:11 cargo*
-rwxr-xr-x 1 alizj 1.1M  5  5 12:11 cargo-clippy*
-rwxr-xr-x 1 alizj 1.5M  5  5 12:11 cargo-fmt*
-rwxr-xr-x 1 alizj  11M  5  5 12:11 clippy-driver*
-rwxr-xr-x 1 alizj  39M  5  5 12:30 rust-analyzer*
-rwxr-xr-x 1 alizj  980  5  5 12:11 rust-gdb*
-rwxr-xr-x 1 alizj 2.2K  5  5 12:11 rust-gdbgui*
-rwxr-xr-x 1 alizj 1.1K  5  5 12:11 rust-lldb*
-rwxr-xr-x 1 alizj 626K  5  5 12:11 rustc*
-rwxr-xr-x 1 alizj  11M  5  5 12:11 rustdoc*
-rwxr-xr-x 1 alizj 6.3M  5  5 12:11 rustfmt*

zj@a:~/docs$ ln -sf ~/.rustup/toolchains/stable-aarch64-apple-darwin/bin/rust-analyzer ~/.rustup/toolchains/esp/bin/rust-analyzer
```

There are the following approaches to using Rust on Espressif chips:

-   Using the std library, a.k.a. `Standard library`.
-   Using `the core library (no_std)`, a.k.a. bare metal development.

In the esp-rs organization, we use the following wording:

-   Repositories starting with `esp-` are focused on `no_std` approach. For example, `esp-hal`
    -   no_std works on top of bare metal, so esp- is an Espressif chip
-   Repositories starting with `esp-idf-` are focused on `std` approach. For example, `esp-idf-hal`
    -   std, apart from bare metal, also needs an additional layer, which is esp-idf-

对于 RISC-V targets，rust 官方是 Tier2 支持，可以直接向官方 rustc 添加对应 target：

-   Newer Espressif chips are all RISC-V based.

<!--listend-->

```shell
rustup toolchain install nightly --component rust-src

# For no_std (bare-metal) applications
rustup target add riscv32imc-unknown-none-elf # For ESP32-C2 and ESP32-C3
rustup target add riscv32imac-unknown-none-elf # For ESP32-C6 and ESP32-H2

# For std applications
# Since this target is currently Tier 3,
# riscv32imc-esp-espidf 和 riscv32imac-esp-espidf
```

esp32 fork 了 rust 工具链，通过 espup 安装了 esp channel 的 rust 工具链，以支持 xtensa CPU：

```shell
# esp rustc 支持 x86_64/arm64/riscv64/xtensa-esp32s3-espidf/xtensa-esp32s3-none-elf target
# 后续可以在项目的 rust-toolchain.toml 和 .cargo/config.toml 中指定使用 esp channel 和对应的 target。
zj@a:~/docs$  ~/.rustup/toolchains/esp/bin/rustc --print target-list |grep -E 'xtensa|riscv'
riscv32gc-unknown-linux-gnu
riscv32gc-unknown-linux-musl
riscv32i-unknown-none-elf
riscv32im-unknown-none-elf
riscv32imac-esp-espidf
riscv32imac-unknown-none-elf
riscv32imac-unknown-xous-elf
riscv32imc-esp-espidf
riscv32imc-unknown-none-elf
riscv64-linux-android
riscv64gc-unknown-freebsd
riscv64gc-unknown-fuchsia
riscv64gc-unknown-hermit
riscv64gc-unknown-linux-gnu
riscv64gc-unknown-linux-musl
riscv64gc-unknown-netbsd
riscv64gc-unknown-none-elf
riscv64gc-unknown-openbsd
riscv64imac-unknown-none-elf
xtensa-esp32-espidf
xtensa-esp32-none-elf
xtensa-esp32s2-espidf
xtensa-esp32s2-none-elf
xtensa-esp32s3-espidf  # espidf 后缀的 target 为使用 eps-idf 的 std 应用
xtensa-esp32s3-none-elf # none-elf 后缀的 target 为不使用 esp-idf 的 non_std 应用
xtensa-esp8266-none-elf
```


## <span class="section-num">4</span> rust std 应用 {#rust-std-应用}

esp-idf 是 C-based 开发框架。esp-idf, in turn, provides `a newlib environment` with enough
functionality to `build the Rust standard library (std) on top of it`. This is the approach that is
being taken to `enable std support` on Epressif devices.

-   esp-idf：<https://github.com/espressif/esp-idf#esp-idf-release-and-soc-compatibility/>
-   可以使用 rust std 库中的丰富特性，如 networking protocols, file I/O, or complex data structures；

std 是通过 <https://github.com/esp-rs/rust> 提供的 rustc 支持的（安装到了 ~/.rustup/toolchains/esp 目录下），后续被链接到 C 接口的 esp-idf framework。

-   <https://github.com/esp-rs/rust/tree/esp-1.76.0.0/src/tools/rust-analyzer>
-   提供了 x.py

<!--listend-->

```shell
# 源码编译
./configure --experimental-targets=Xtensa --release-channel=nightly --enable-extended
  --tools=clippy,cargo,rustfmt --enable-lld
# 可以在 --tools 中指定 rust-analyzer 来编译安装 esp32 使用的 rust-analyzer
```

std 相关的库：

embedded-svc
: Abstraction traits for embedded services (WiFi, Network, Httpd, Logging, etc.)

esp-idf-svc
: An implementation of embedded-svc using esp-idf drivers.

esp-idf-hal
: An implementation of the embedded-hal and other traits using the esp-idf framework.

esp-idf-sys
: Rust bindings to the esp-idf development framework. Gives raw (unsafe) access to
    drivers, Wi-Fi and more.

相关解释：

-   esp-idf：esp 的 C 接口框架；
-   esp-idf-sys 是 esp-idf 框架的 rust binding，非安全接口风格，是 std 的基础。
    -   一般不直接使用 esp-dif-sys，而是使用在它基础上封装的， rust 安全接口 的 esp-idf-hal 和 esp-idf-svc。
-   esp-idf-hal 是 embedded-hal trait 的实现；
-   esp-idf-svc 是 embedded-svc trait 的实现，和 hal 相比主要是一些上层服务。

如果 esp-dif-hal 不满足需求（如缺少一些 esp32 的寄存器的操作），可以使用 esp-rs/esp-pacs 下的
[esp32s3 create](https://github.com/esp-rs/esp-pacs/tree/main/esp32s3), 它是使用 svd2rust 工具来基于芯片的 svd 自动生成的库。

<https://apollolabsblog.hashnode.dev/the-embedded-rust-esp-development-ecosystem>

{{< figure src="/images/esp-idf_和_std_应用/2024-02-11_15-26-13_screenshot.png" width="400" >}}

一般使用 esp-rs/esp-idf-template 模板项目来创建 std 项目：

cargo generate esp-rs/esp-idf-template cargo
: 使用 cargo 来构建纯 rust 应用（cargo-frst）；

cargo generate esp-rs/esp-idf-template cmake
: mix Rust and C/C++ in a traditional ESP-IDF idf.py；

<!--listend-->

```shell
# STD Project
cargo generate esp-rs/esp-idf-template cargo  # 除了 cargo 外，还可以选择 cmake
# NO-STD (Bare-metal) Project
cargo generate esp-rs/esp-template
```

1.  cargo-first 项目：
2.  <https://github.com/esp-rs/esp-idf-template/blob/master/README.md>
3.  STD support：When true, adds support for the Rust Standard Library. Otherwise, we will use Rust
    Core Library.
4.  在 .cargo/config.toml 的 env 部分添加：ESP_IDF_TOOLS_INSTALL_DIR = "global"

<!--listend-->

```shell
zj@a:~/codes/esp32/$ cargo generate esp-rs/esp-idf-template cargo
⚠️   Favorite `esp-rs/esp-idf-template` not found in config, using it as a git repository: https://github.com/esp-rs/esp-idf-template.git
🔧   project-name: myesp ...
🔧   Generating template ...
✔ 🤷   Which MCU to target? · esp32s3
✔ 🤷   Configure advanced template options? · true
✔ 🤷   Enable STD support? · true
✔ 🤷   Configure project to use Dev Containers (VS Code and GitHub Codespaces)? · false
✔ 🤷   Configure project to support Wokwi simulation with Wokwi VS Code extension? · false
✔ 🤷   Add CI files for GitHub Action? · false
✔ 🤷   ESP-IDF version (master = UNSTABLE) · v5.1
🔧   Moving generated files into: `/Users/zhangjun/codes/esp32/esp-demo2/myesp`...
🔧   Initializing a fresh Git repository
✨   Done! New project created /Users/zhangjun/codes/esp32/esp-demo2/myesp
zj@a:~/codes/esp32/$ cd myesp

# 修改 .cargo/config.toml 文件中的 ESP_IDF_VERSION 为最新版本，内容如下：
zj@a:~/codes/esp32/myesp$ cat .cargo/config.toml
[build]
target = "xtensa-esp32s3-espidf"  # 要构建的 target，这里使用链接 esp-idf 的 target

[target.xtensa-esp32s3-espidf]  # target 对应的配置
linker = "ldproxy" # 位于 ~/.cargo/bin/
# runner = "espflash --monitor" # Select this runner for espflash v1.x.x
runner = "espflash flash --monitor" # Select this runner for espflash v2.x.x
rustflags = [ "--cfg",  "espidf_time64"] # Extending time_t for ESP IDF 5: https://github.com/esp-rs/rust/issues/110

[unstable]
build-std = ["std", "panic_abort"]

[env]  # 被 embuild 使用的环境变量
MCU="esp32s3"
# Note: this variable is not used by the pio builder (`cargo build --features pio`)
ESP_IDF_VERSION = "v5.2.1"
# 没有配置 ESP_IDF_TOOLS_INSTALL_DIR = "global" 来使用全局 ~/.espressif/ 工具链，默认是 by 项目 workspace 的。

# 构建项目：每次构建前都需要先 source ~/esp/export-esp.sh 脚本。
# 不能启用 python env，不能使用 socks 代理，需要设置环境变量
zj@a:~/code/esp32/myesp$ source ~/esp/export-esp.sh
zj@a:~/code/esp32/myesp$ cargo build

zj@a:~/code/esp32/myesp$ ls target/
CACHEDIR.TAG  debug/  xtensa-esp32s3-espidf/
zj@a:~/code/esp32/myesp$ ls -l target/xtensa-esp32s3-espidf/debug/
total 11M
-rw-r--r--   1 alizj  21K  5  5 14:45 bootloader.bin # bootloader
drwxr-xr-x  18 alizj  576  5  5 14:39 build/
drwxr-xr-x 178 alizj 5.6K  5  5 14:45 deps/
drwxr-xr-x   2 alizj   64  5  5 14:39 examples/
drwxr-xr-x   3 alizj   96  5  5 14:45 incremental/
-rwxr-xr-x   1 alizj  11M  5  5 14:45 myesp*  # 二进制程序
-rw-r--r--   1 alizj  153  5  5 14:45 myesp.d
-rw-r--r--   1 alizj 3.0K  5  5 14:45 partition-table.bin # 分区表
zj@a:~/code/esp32/myesp$ file target/xtensa-esp32s3-espidf/debug/myesp
target/xtensa-esp32s3-espidf/debug/myesp: ELF 32-bit LSB executable, Tensilica Xtensa, version 1 (SYSV), statically linked, with debug_info, not stripped
```

cargo generate 和 cargo build 的结果：

```shell
# esp32 rust 项目通过 rust-toolchain.toml 来选择 channel 和 target
zj@a:~/code/esp32/myespv2$ cat rust-toolchain.toml
[toolchain]
channel = "esp"  # ~/.rustup/toolchains/ 下的目录名称，这里使用 esp toolchain

# 使用 embuild crate 来安装和构建 esp-idf framework
# 对于 non_std 应用，不依赖 esp-idf， 故不需要 build.rs .
zj@a:~/code/esp32/myespv2$ cat build.rs
fn main() {
    embuild::espidf::sysenv::output();
}

# by workspace 安装的 esp-idf 到 .embuild/espressif/ 目录
zj@a:~/code/esp32/myespv2$ ls -l .embuild/espressif/
total 4.0K
drwxr-xr-x 6 alizj  192  5  5 14:54 dist/
drwxr-xr-x 3 alizj   96  5  5 14:49 esp-idf/
-rw-r--r-- 1 alizj 2.8K  5  5 14:51 espidf.constraints.v5.2.txt
drwxr-xr-x 3 alizj   96  5  5 14:51 python_env/
drwxr-xr-x 6 alizj  192  5  5 14:54 tools/

# dist 下载的内容被解压到 .embuild/espressif/tools/ 目录
zj@a:~/code/esp32/myespv2$ ls -l .embuild/espressif/dist/
total 193M
-rw-r--r-- 1 alizj  70M  5  5 14:52 cmake-3.24.0-macos-universal.tar.gz
-rw-r--r-- 1 alizj  15M  5  5 14:54 esp32ulp-elf-2.35_20220830-macos-arm64.tar.gz
-rw-r--r-- 1 alizj 271K  5  5 14:52 ninja-mac-v1.11.1.zip
-rw-r--r-- 1 alizj  96M  5  5 14:51 xtensa-esp-elf-13.2.0_20230928-aarch64-apple-darwin.tar.xz # 交叉编译工具链

zj@a:~/code/esp32/myespv2$ ls -l .embuild/espressif/tools/
total 0
drwxr-xr-x 3 alizj 96  5  5 14:52 cmake/
drwxr-xr-x 3 alizj 96  5  5 14:54 esp32ulp-elf/ # ULP (Ultra-Low-Powered)
drwxr-xr-x 3 alizj 96  5  5 14:52 ninja/
drwxr-xr-x 3 alizj 96  5  5 14:51 xtensa-esp-elf/

# esp-idf framework，被安装到 python_env/ 目录
zj@a:~/code/esp32/myespv2$ ls -l .embuild/espressif/esp-idf/v5.2.1/
total 172K
-rw-r--r--  1 alizj  12K  5  5 14:49 CMakeLists.txt
-rw-r--r--  1 alizj 4.2K  5  5 14:49 COMPATIBILITY.md
-rw-r--r--  1 alizj 4.3K  5  5 14:49 COMPATIBILITY_CN.md
-rw-r--r--  1 alizj  314  5  5 14:49 CONTRIBUTING.md
-rw-r--r--  1 alizj  25K  5  5 14:49 Kconfig
-rw-r--r--  1 alizj  12K  5  5 14:49 LICENSE
-rw-r--r--  1 alizj 8.9K  5  5 14:49 README.md
-rw-r--r--  1 alizj 8.8K  5  5 14:49 README_CN.md
-rw-r--r--  1 alizj  532  5  5 14:49 SECURITY.md
-rw-r--r--  1 alizj 3.7K  5  5 14:49 SUPPORT_POLICY.md
-rw-r--r--  1 alizj 3.4K  5  5 14:49 SUPPORT_POLICY_CN.md
-rw-r--r--  1 alizj  721  5  5 14:49 add_path.sh
drwxr-xr-x 81 alizj 2.6K  5  5 14:49 components/
-rw-r--r--  1 alizj  12K  5  5 14:49 conftest.py
drwxr-xr-x 15 alizj  480  5  5 14:49 docs/
drwxr-xr-x 22 alizj  704  5  5 14:49 examples/
-rw-r--r--  1 alizj 3.9K  5  5 14:49 export.bat
-rw-r--r--  1 alizj 3.7K  5  5 14:49 export.fish
-rw-r--r--  1 alizj 3.5K  5  5 14:49 export.ps1
-rw-r--r--  1 alizj 8.0K  5  5 14:49 export.sh
-rw-r--r--  1 alizj 1.8K  5  5 14:49 install.bat
-rwxr-xr-x  1 alizj  971  5  5 14:49 install.fish*
-rw-r--r--  1 alizj  982  5  5 14:49 install.ps1
-rwxr-xr-x  1 alizj 1004  5  5 14:49 install.sh*
-rw-r--r--  1 alizj  889  5  5 14:49 pytest.ini
-rw-r--r--  1 alizj 2.0K  5  5 14:49 sdkconfig.rename
-rw-r--r--  1 alizj  530  5  5 14:49 sonar-project.properties
drwxr-xr-x 47 alizj 1.5K  5  5 14:51 tools/
zj@a:~/code/esp32/myespv2$
```

构建失败的解决办法：

1.  清理 target 和 .embuild 目录；
2.  不能启用 socks 代理；
3.  不能开启 python venv；

<!--listend-->

```shell
zj@a:~/code/esp32/myesp$ cargo clean
zj@a:~/code/esp32/myesp$ rm -rf .embuild/
zj@a:~/code/esp32/myesp$ ls
Cargo.lock  Cargo.toml  build.rs  rust-toolchain.toml  sdkconfig.defaults  src/
zj@a:~/code/esp32/myesp$ cargo build

# cargo build 会安装 esp-idf，期间会安装 python venv 和按照 python 包。所以，不能使用 python 不支持
# 的 socks 代理，也不能启用 python env。
zj@a:~/code/esp32/$ enable_http_proxy
zj@a:~/code/esp32/$ export DIR_TO_REMOVE=/Users/alizj/.venv/bin
zj@a:~/code/esp32/$ export PATH=$(echo $PATH | sed -e "s;:$DIR_TO_REMOVE;;" -e "s;$DIR_TO_REMOVE:;;" -e "s;$DIR_TO_REMOVE;;")

# 如果是 Mac M1 笔记本，需要给 cargo build 添加环境变量 CRATE_CC_NO_DEFAULTS=1，否则会构建失败，报错：
# xtensa-esp-elf-gcc: error: unrecognized command-line option '--target=xtensa-esp32s3-espidf'
# 参考：https://github.com/rust-lang/cc-rs/issues/1005
zj@a:~/code/esp32/$ export CRATE_CC_NO_DEFAULTS=1
```

1.  cmake 项目：
2.  <https://github.com/esp-rs/esp-idf-template/blob/master/README-cmake.md>
3.  原理：<https://github.com/esp-rs/esp-idf-template/blob/master/README-cmake-details.md>
4.  需要先手动安装 esp-idf，然后 source ~/esp/esp-idf/export.sh 脚本;
    -   也可以使用 cargo build 自动安装的 ~/.espressif/esp-idf/v5.2/export.sh 脚本;
5.  需要在项目的 components/rust-{{project-name}}/CMakeLists.txt 文件中 Rust 应用依赖的 component。
    -   默认只添加了 "pthread" "driver" "vfs"，其他可选："pthread" "esp_http_client" "esp_http_server"
        "espcoredump" "app_update" "esp_serial_slave_link" "nvs_flash" "spi_flash" "esp_adc_cal" "mqtt"；

<!--listend-->

```shell
# 创建项目
zj@a:~/codes/esp32/esp-demo$ enable_http_proxy
zj@a:~/codes/esp32/esp-demo2$ cargo generate esp-rs/esp-idf-template cmake

# 构建
cd <your-project-name>
source ~/esp/esp-idf/export.sh
enable_http_proxy
idf.py set-target [esp32|esp32s2|esp32s3|esp32c2|esp32c3|esp32c6|esp32h2]
idf.py build

# Flash
idf.py -p /dev/ttyUSB0 flash

# Monitor
idf.py -p /dev/ttyUSB0 monitor
```

对于使用 esp-idf-sys 的 cargo-first std 应用，项目目录下的 `build.rs` 文件会使用 embuild crate 来下载、编译和链接 esp-idf C framework 和 gcc toolchian。

-   默认是 by 项目下载 esp-idf 的，为了加快速度，可以设置 `ESP_IDF_TOOLS_INSTALL_DIR=global` 来使用

全局 toolchain（~/.espressif/esp-idf/&lt;version&gt;）。

使用 cargo 构建 eps-idf-sys 时 embuild 会读取配置：
<https://github.com/esp-rs/esp-idf-sys/blob/master/BUILD-OPTIONS.md>

1.  .cargo/config.toml 中的传递给 rustc 的 flags；
2.  环境变量：
    1.  cargo/config.toml env section 中的环境变量（如 ESP_IDF_SDKCONFIG_DEFAULTS，大写） ；（优先级最高）
    2.  项目根目录下的 Cargo.toml 文件中的 [package.metadata.esp-idf-sys] section 的配置参数（小写），如
        esp_idf_sdkconfig_defaults；
3.  项目的 sdkconfig 文件：
    <https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/kconfig.html#project-configuration>

配置示例（项目 Cargo.toml 文件）：

```toml
[package.metadata.esp-idf-sys]
esp_idf_tools_install_dir = "global"
esp_idf_sdkconfig = "sdkconfig"
esp_idf_sdkconfig_defaults = ["sdkconfig.defaults", "sdkconfig.defaults.ble"]
# native builder only
esp_idf_version = "branch:release/v4.4"
esp_idf_components = ["pthread"]
```

可以设置的环境变量列表：<https://github.com/esp-rs/esp-idf-sys/blob/master/BUILD-OPTIONS.md>

1.  esp_idf_sdkconfig_defaults, $ESP_IDF_SDKCONFIG_DEFAULTS
    -   默认为 sdkconfig.defaults;
2.  esp_idf_sdkconfig, $ESP_IDF_SDKCONFIG
    -   默认为 sdkconfig
3.  esp_idf_tools_install_dir, $ESP_IDF_TOOLS_INSTALL_DIR
    -   可选值为：
        -   workspace（缺省），默认为 &lt;crate-workspace-dir&gt;/.embuild/espressif；
        -   out - the tooling will be installed or used inside esp-idf-sys's build output directory,
            and will be deleted when `cargo clean` is invoked;
        -   `global` - the tooling will be installed or used in its standard directory (~/.platformio
            for PlatformIO, and `~/.espressif` for the native ESP-IDF toolset);
        -   custom:&lt;dir&gt; - the tooling will be installed or used in the directory specified by
            &lt;dir&gt;. If this directory is a relative location, it is assumed to be relative to the
            workspace directory;
4.  idf_path, $IDF_PATH (native builder only)
    -   A path to a user-provided local clone of the esp-idf, that will be used instead of the one
        downloaded by the build script.
5.  esp_idf_version, $ESP_IDF_VERSION (native builder only)
    -   The version used for the esp-idf, can be one of the following:
6.  mcu, $MCU
    -   The MCU name (i.e. esp32, esp32s2, esp32s3 esp32c3, esp32c2, esp32h2, esp32c5, esp32c6, esp32p4).
7.  esp_idf_components, $ESP_IDF_COMPONENTS (native builder only)
    -   Defaults to all components being built.

为了加快 cargo build 速率，避免每次都重新编译构建 esp-idf，建议使用 `.cargo/config.toml` 示例配置(在项目执行 cargo build 命令来进行验证):

```toml
[build]
target = "xtensa-esp32s3-espidf"
# 使用 sccache 来调用 rustc 编译器, 有利用缓存加快构建速度.
# 先安装 sccache: cargo install sccache --locked
rustc-wrapper = "/opt/homebrew/bin/sccache"

[target.xtensa-esp32s3-espidf]
linker = "ldproxy"
# runner = "espflash --monitor" # Select this runner for espflash v1.x.x
runner = "espflash flash --monitor" # Select this runner for espflash v2.x.x

# 以下是使用 rustc 构建 esp-rs/esp-idf-sys 时传递的参数
# https://github.com/esp-rs/esp-idf-sys/blob/master/BUILD-OPTIONS.md This is a flag for the libc
# crate that uses 64-bits (instead of 32-bits) for time_t. This must be set for ESP-IDF 5.0 and
# above and must be unset for lesser versions.
rustflags = [ "--cfg",  "espidf_time64"] # Extending time_t for ESP IDF 5: https://github.com/esp-rs/rust/issues/110
# https://github.com/esp-rs/esp-idf-sys/blob/master/BUILD-OPTIONS.md
# 等效于： -Zbuild-std=std,panic_abort
# Required for std support. Rust does not provide std libraries for ESP32 targets since they are tier-2/-3.
[unstable]
build-std = ["std", "panic_abort"]

# cargo 调用命令时使用的环境变量
# 参考：https://github.com/esp-rs/esp-idf-sys/blob/master/BUILD-OPTIONS.md#esp-idf-configuration
[env]
MCU="esp32s3"
# Note: this variable is not used by the pio builder (`cargo build --features pio`)
# 最新 esp-idf 版本：https://github.com/espressif/esp-idf/releases
ESP_IDF_VERSION = "v5.2.q"

# 使用全局 ~/.espressif/ 工具链，默认是 by 项目 workspace 的。
ESP_IDF_TOOLS_INSTALL_DIR = "global"
```

sccache 统计信息:

```shell
zj@a:~/code/esp32/myespv3$ sccache --show-stats
Compile requests                      0
Compile requests executed             0
Cache hits                            0
Cache misses                          0
Cache timeouts                        0
Cache read errors                     0
Forced recaches                       0
Cache write errors                    0
Compilation failures                  0
Cache errors                          0
Non-cacheable compilations            0
Non-cacheable calls                   0
Non-compilation calls                 0
Unsupported compiler calls            0
Average cache write               0.000 s
Average compiler                  0.000 s
Average cache read hit            0.000 s
Failed distributed compilations       0
Cache location                  Local disk: "/Users/alizj/Library/Caches/Mozilla.sccache"
Use direct/preprocessor mode?   yes
Version (client)                0.7.7
Max cache size                       10 GiB
```

std 参考示例

1.  [esp-rs/std-trainning](https://github.com/esp-rs/std-training): Embedded Rust Trainings for Espressif
2.  <https://github.com/ivmarkov/rust-esp32-std-demo%EF%BC%9A> Rust on ESP32 STD demo app
3.  <https://apollolabsblog.hashnode.dev/series/esp32-std-embedded-rust%EF%BC%9A> 强烈推荐。
    -   <https://github.com/apollolabsdev/ESP32C3>


## <span class="section-num">5</span> rust no_std 应用 {#rust-no-std-应用}

对于 no_std 则使用 Rust core 库，core 是 std 库的一个子集。 当前支持：
HAL/WIFI/BLE/ESP-NOW/Backtrace/Storage。

参考：[Bare-Metal Rust on ESP32: A Brief Overview](https://beta7.io/posts/bare-metal-rust-on-esp32/)

<https://apollolabsblog.hashnode.dev/the-embedded-rust-esp-development-ecosystem>

{{< figure src="/images/no_std_应用/2024-02-11_15-27-15_screenshot.png" width="400" >}}

no_std 相关的库：

-   esp-hal	Hardware abstraction layer
-   esp-pacs	Peripheral access crates
-   esp-wifi	Wi-Fi, BLE and ESP-NOW support
-   esp-alloc	Simple heap allocator
-   esp-println print!, println!
-   esp-backtrace Exception and panic handlers
-   esp-storage Embedded-storage traits to access unencrypted flash memory

一般使用 esp-rs/esp-template 模板来快速创建 non_std 类型项目：

-   在 .cargo/config.toml 的 env 部分添加 ESP_IDF_TOOLS_INSTALL_DIR = "global"
-   可以指定是否使用 WiFi/Bluetooth/ESP-NOW via the esp-wifi crate；

<!--listend-->

```shell
# STD Project
cargo generate esp-rs/esp-idf-template cargo
# NO-STD (Bare-metal) Project
cargo generate esp-rs/esp-template

# 创建一个项目
zj@a:~/code/esp32$ cargo generate esp-rs/esp-template
# 构建
zj@a:~/code/esp32$ cd myesp-nonstd/
zj@a:~/code/esp32/myesp-nonstd$ source ~/esp/export-esp.sh
zj@a:~/code/esp32/myesp-nonstd$ cat rust-toolchain.toml
[toolchain]
channel = "esp" # 使用 esp channel 工具链
zj@a:~/code/esp32/myesp-nonstd$ cat .cargo/config.toml
[target.xtensa-esp32s3-none-elf]
runner = "espflash flash --monitor"


[env]
ESP_LOGLEVEL="INFO"

[build]
rustflags = [
  "-C", "link-arg=-nostartfiles",
]

target = "xtensa-esp32s3-none-elf" # 使用不链接 esp-idf 的 none-elf 工具链

[unstable]
build-std = ["core"]
zj@a:~/code/esp32/myesp-nonstd$ cargo build
```

参考示例

1.  <https://apollolabsblog.hashnode.dev/series/esp32c3-embedded-rust-hal> 强烈推荐。
2.  <https://github.com/apollolabsdev/ESP32C3>


## <span class="section-num">6</span> espflash 烧录和监视工具 {#espflash-烧录和监视工具}

espflash 使用 USB 串口(linux: /dev/ttyUSB0, macOS: /dev/cu.\*) 来烧录芯片:

-   <https://github.com/esp-rs/espflash/tree/main/espflash#installation>

<!--listend-->

```shell
zj@a:~/codes/esp32$ espflash --help
A command-line tool for flashing Espressif devices

Usage: espflash <COMMAND>

Commands:
  board-info       Print information about a connected target device
  completions      Generate completions for the given shell
  erase-flash      Erase Flash entirely
  erase-parts      Erase specified partitions
  erase-region     Erase specified region
  flash            Flash an application in ELF format to a connected target device
  monitor          Open the serial monitor without flashing the connected target device
  partition-table  Convert partition tables between CSV and binary format
  save-image       Generate a binary application image and save it to a local disk
  write-bin        Write a binary file to a specific address in a target device's flash
  help             Print this message or the help of the given subcommand(s)

Options:
  -h, --help     Print help
  -V, --version  Print version
zj@a:~/codes/esp32$
```

cargo run: 在项目的 .cargo/config.toml 中添加如下内容, 然后就可以执行 cargo run 来 flash 和 monitor 应用:

```rust
[target.'cfg(any(target_arch = "riscv32", target_arch = "xtensa"))']
runner = "espflash flash --baud=921600 --monitor /dev/ttyUSB0"
```

espflash 配置文件 espflash.toml:

1.  Serial port:
    ```toml
    [connection]
    serial = "/dev/ttyUSB0"
    ```
2.  baudrate = 460800
3.  bootloader = "path/to/custom/bootloader.bin"
4.  partition_table = "path/to/custom/partition-table.bin"

espflash.toml 文件位置:

1.  当前目录;
2.  $HOME/Library/Application Support/rs.esp.espflash/espflash.toml

Establishing a serial connection with the ESP32-S3 target device could be done using `USB-to-UART
bridge` or `USB peripheral` supported in ESP32-S3.

For the ESP32-S3, `the USB peripheral` is available, allowing you to flash the binaries `without the
need for an external USB-to-UART bridge`.

The USB on the ESP32-S3 uses the GPIO20 for D+ and GPIO19 for D-.

If you are flashing for the first time, you need to get the ESP32-S3 into `the download mode`
manually. To do so, press and hold the BOOT button and then press the RESET button once. After that
release the BOOT button.


## <span class="section-num">7</span> 报错 {#报错}

1.  在 esp-idf-template 生成的模板项目中执行 cargo build 报错:
2.  <https://github.com/esp-rs/esp-idf-template/issues/165>

<!--listend-->

```rust
  Using managed esp-idf repository: RemoteSdk { repo_url: None, git_ref: Tag("v5.1.2") }
  Using esp-idf v5.1.2 at '/Users/zhangjun/codes/esp32/esp-demo/.embuild/espressif/esp-idf/v5.1.2'
  ERROR: /Users/zhangjun/codes/esp32/esp-demo/.embuild/espressif/espidf.constraints.v5.1.txt doesn't exist. Perhaps you've forgotten to run the install scripts. Please check the installation guide for more information.
  CMake Error at /Users/zhangjun/codes/esp32/esp-demo/.embuild/espressif/esp-idf/v5.1.2/tools/cmake/build.cmake:363 (message):
    Some Python dependencies must be installed.  Check above message for
    details.
  Call Stack (most recent call first):
    /Users/zhangjun/codes/esp32/esp-demo/.embuild/espressif/esp-idf/v5.1.2/tools/cmake/build.cmake:498 (__build_check_python)
    /Users/zhangjun/codes/esp32/esp-demo/.embuild/espressif/esp-idf/v5.1.2/tools/cmake/project.cmake:547 (idf_build_process)
    CMakeLists.txt:28 (project)


  thread 'main' panicked at /Users/zhangjun/.cargo/registry/src/index.crates.io-6f17d22bba15001f/cmake-0.1.50/src/lib.rs:1098:5:

  command did not execute successfully, got: exit status: 1

  build script failed, must exit now
  note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

解决办法:

```shell
cargo clean && rm -rf .embuild && cargo build
```

1.  报错: Missing dependencies for SOCKS support.

<!--listend-->

```shell
  来自 https://github.com/ThrowTheSwitch/Unity
   * branch            7d2bf62b7e6afaf38153041a9d53c21aeeca9a25 -> FETCH_HEAD
  ERROR: Could not install packages due to an OSError: Missing dependencies for SOCKS support.

  WARNING: There was an error checking the latest version of pip.
  Error: Command '['/Users/zhangjun/codes/esp32/esp-demo/.embuild/espressif/python_env/idf5.1_py3.12_env/bin/python3', '-m', 'pip', 'install', '--upgrade', 'pip']' returned non-zero exit status 1.
  Traceback (most recent call last):
    File "/Users/zhangjun/codes/esp32/esp-demo/.embuild/espressif/esp-idf/v5.1.2/tools/idf_tools.py", line 2687, in <module>
      main(sys.argv[1:])
    File "/Users/zhangjun/codes/esp32/esp-demo/.embuild/espressif/esp-idf/v5.1.2/tools/idf_tools.py", line 2679, in main
      action_func(args)
    File "/Users/zhangjun/codes/esp32/esp-demo/.embuild/espressif/esp-idf/v5.1.2/tools/idf_tools.py", line 2098, in action_install_python_env
      subprocess.check_call([sys.executable, '-m', 'venv',
    File "/Users/zhangjun/.pyenv/versions/3.12.1/lib/python3.12/subprocess.py", line 413, in check_call
      raise CalledProcessError(retcode, cmd)
  subprocess.CalledProcessError: Command '['/Users/zhangjun/.pyenv/versions/3.12.1/bin/python3', '-m', 'venv', '--clear', '--upgrade-deps', '/Users/zhangjun/codes/esp32/esp-demo/.embuild/espressif/python_env/idf5.1_py3.12_env']' returned non-zero exit status 1.
  Error: Could not install esp-idf

  Caused by:
      command 'env -u IDF_PYTHON_ENV_PATH -u MSYSTEM IDF_TOOLS_PATH="/Users/zhangjun/codes/esp32/esp-demo/.embuild/espressif" "python3" "/Users/zhangjun/codes/esp32/esp-demo/.embuild/espressif/esp-idf/v5.1.2/tools/idf_tools.py" "--idf-path" "/Users/zhangjun/codes/esp32/esp-demo/.embuild/espressif/esp-idf/v5.1.2" "--non-interactive" "install-python-env"' exited with non-zero status code 1
zj@a:~/codes/esp32/esp-demo$
```

解决办法: 不使用 socks5 代理, 而是使用 https/http 代理:

```shell
# 将下列内容添加到 ~/esp/export.sh 和 ~/esp/export-esp.sh 中
export all_proxy="http://192.168.3.2:1080" ALL_PROXY="http://192.168.3.2:1080"

# 在 ~/.bashrc 中添加如下内容， 用于手动切换：
alias enable_http_proxy='export all_proxy="http://192.168.3.2:1080" ALL_PROXY="http://192.168.3.2:1080"'
alias enable_socks_proxy='export all_proxy="socks5h://192.168.3.2:1080" ALL_PROXY="socks5h://192.168.3.2:1080"'
alias disable_proxy='unset all_proxy ALL_PROXY'
```

1.  rust-analyzer 报错，不能正常解析和补全。解决办法：
    ```nil
    zj@a:~/codes/esp32/esp-demo2/myesp$ cd

    zj@a:~$ rustup component add rust-analyzer
    info: downloading component 'rust-analyzer'
    info: installing component 'rust-analyzer'

    zj@a:~$ ls -l ~/.rustup/toolchains/
    esp/                         nightly-x86_64-apple-darwin/

    zj@a:~$ ls -l ~/.rustup/toolchains/nightly-x86_64-apple-darwin/bin/
    total 95M
    -rwxr-xr-x 1 zhangjun  29M  2  8 14:48 cargo*
    -rwxr-xr-x 1 zhangjun 1.1M  2  8 14:48 cargo-clippy*
    -rwxr-xr-x 1 zhangjun 1.5M  2  8 14:49 cargo-fmt*
    -rwxr-xr-x 1 zhangjun  11M  2  8 14:48 clippy-driver*
    -rwxr-xr-x 1 zhangjun  36M  2  8 16:57 rust-analyzer*
    -rwxr-xr-x 1 zhangjun  980  2  8 14:48 rust-gdb*
    -rwxr-xr-x 1 zhangjun 2.2K  2  8 14:49 rust-gdbgui*
    -rwxr-xr-x 1 zhangjun 1.1K  2  8 14:48 rust-lldb*
    -rwxr-xr-x 1 zhangjun 598K  2  8 14:49 rustc*
    -rwxr-xr-x 1 zhangjun  11M  2  8 14:49 rustdoc*
    -rwxr-xr-x 1 zhangjun 6.6M  2  8 14:49 rustfmt*

    zj@a:~$ ln -sf ~/.rustup/toolchains/nightly-x86_64-apple-darwin/bin/rust-analyzer ~/.rustup/toolchains/esp/bin/rust-analyzer
    z
    ```
