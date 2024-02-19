---
title: "Rust ESP32 开发"
author: ["张俊(zj@opsnull.com)"]
date: 2024-02-19T00:00:00+08:00
lastmod: 2024-02-19T21:52:14+08:00
tags: ["rust", "esp32"]
categories: ["rust", "esp32"]
draft: false
series: ["rust-esp32"]
series_order: 1
---

## <span class="section-num">1</span> esp-rs {#esp-rs}

eps-rs 研发计划：<https://github.com/orgs/esp-rs/projects/2>

ESP32 Rust 开发环境：
<https://apollolabsblog.hashnode.dev/the-embedded-rust-esp-development-ecosystem>

各种 ESP32 芯片开发语言的[比较](https://github.com/georgik/esp32-lang-lab)：

{{< figure src="/images/esp-rs/2024-02-11_15-51-29_screenshot.png" width="400" >}}

参考示例代码：

1.  <https://github.com/apollolabsdev/ESP32C3%EF%BC%9A>。


## <span class="section-num">2</span> 安装 esp-idf {#安装-esp-idf}

除了使用 c/c++ 原生方式使用 esp-idf 外，在进行 Rust 和 c/++ 混合编程时（如 cargo generate
esp-rs/esp-idf-template cmake) 时也使用 esp-idf 的开发、构建和烧写工具 idf.py.

-   <https://github.com/esp-rs/esp-idf-template/blob/master/README-cmake.md>

[
安装 ESP-IDF](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/linux-macos-setup.html#for-macos-users) 到 ~/.espressif/:

```shell
$ brew install cmake ninja dfu-util
$ brew install ccache # 可选, 加快构建速度
$ echo 'PATH=/local/opt/ccache/libexec:$PATH' >>~/.bashrc
$ python --version  # 确保系统是 python 3 版本

$ mkdir -p ~/esp
$ cd ~/esp
$ git clone --recursive https://github.com/espressif/esp-idf.git
$ cd ~/esp/esp-idf

$ enable_http_proxy  # python 不支持 SOCKS5 代理，否则执行下面脚本会出错。
$ ./install.sh esp32s3  #  esp32,esp32s2 等目标芯片, all 表示所有.
Detecting the Python interpreter
Checking "python3" ...
Python 3.12.1
"python3" has been detected
Checking Python compatibility
Installing ESP-IDF tools
Selected targets are: esp32s3
Current system platform: macos
Installing tools: xtensa-esp-elf-gdb, xtensa-esp-elf, riscv32-esp-elf, esp32ulp-elf, openocd-esp32, esp-rom-elfs
Skipping xtensa-esp-elf-gdb@12.1_20231023 (already installed)
Skipping xtensa-esp-elf@esp-13.2.0_20230928 (already installed)
Skipping riscv32-esp-elf@esp-13.2.0_20230928 (already installed)
Skipping esp32ulp-elf@2.35_20220830 (already installed)
Skipping openocd-esp32@v0.12.0-esp32-20230921 (already installed)
Skipping esp-rom-elfs@20230320 (already installed)
Installing Python environment and packages
Python 3.12.1
pip 23.2.1 from /Users/zhangjun/.espressif/python_env/idf5.3_py3.12_env/lib/python3.12/site-packages/pip (python 3.12)
Upgrading pip and setuptools...
Requirement already satisfied: pip in /Users/zhangjun/.espressif/python_env/idf5.3_py3.12_env/lib/python3.12/site-packages (23.2.1)
Successfully installed pip-24.0 setuptools-69.0.3
Downloading https://dl.espressif.com/dl/esp-idf/espidf.constraints.v5.3.txt
Destination: /Users/zhangjun/.espressif/espidf.constraints.v5.3.txt.tmp
Done
Installing Python packages
 Constraint file: /Users/zhangjun/.espressif/espidf.constraints.v5.3.txt
 Requirement files:
  - /Users/zhangjun/esp/esp-idf/tools/requirements/requirements.core.txt
...
Installing collected packages: reedsolo, pyserial, pygdbmi, pyelftools, intelhex, bitarray, urllib3, tqdm, six, pyyaml, pyparsing, pygments, pycparser, pyclang, packaging, msgpack, mdurl, kconfiglib, idna, freertos_gdb, filelock, esp-idf-panic-decoder, contextlib2, construct, colorama, click, charset-normalizer, certifi, bitstring, schema, requests, markdown-it-py, esp-idf-kconfig, ecdsa, cffi, rich, requests-toolbelt, requests-file, cryptography, cachecontrol, esptool, esp-idf-size, esp-idf-nvs-partition-gen, idf-component-manager, esp-coredump, esp-idf-monitor
Successfully installed bitarray-2.9.2 bitstring-4.1.4 cachecontrol-0.14.0 certifi-2024.2.2 cffi-1.16.0 charset-normalizer-3.3.2 click-8.1.7 colorama-0.4.6 construct-2.10.70 contextlib2-21.6.0 cryptography-42.0.2 ecdsa-0.18.0 esp-coredump-1.10.0 esp-idf-kconfig-2.1.0 esp-idf-monitor-1.4.0 esp-idf-nvs-partition-gen-0.1.1 esp-idf-panic-decoder-1.0.1 esp-idf-size-1.1.1 esptool-4.8.dev1 filelock-3.13.1 freertos_gdb-1.0.2 idf-component-manager-1.4.2 idna-3.6 intelhex-2.3.0 kconfiglib-14.1.0 markdown-it-py-3.0.0 mdurl-0.1.2 msgpack-1.0.7 packaging-23.2 pyclang-0.4.2 pycparser-2.21 pyelftools-0.30 pygdbmi-0.11.0.0 pygments-2.17.2 pyparsing-3.1.1 pyserial-3.5 pyyaml-6.0.1 reedsolo-1.7.0 requests-2.31.0 requests-file-1.5.1 requests-toolbelt-1.0.0 rich-13.7.0 schema-0.7.5 six-1.16.0 tqdm-4.66.1 urllib3-1.26.18
All done! You can now run:

  . ./export.sh
```

安装的内容位于 ~/.espressif 目录下：

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
zj@a:~$ which xtensa-esp-elf-gcc
/Users/zhangjun/.espressif/tools/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp-elf-gcc
zj@a:~/codes/esp32/esp-demo2/myesp$ which xtensa-esp32s3-elf-gcc
/Users/zhangjun/.espressif/tools/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-gcc

zj@a:~/codes/esp32/esp-demo2/myesp$ which xtensa-esp32s3-elf-gdb  # 来源于 xtensa-esp-elf-gdb
/Users/zhangjun/.espressif/tools/xtensa-esp-elf-gdb/12.1_20231023/xtensa-esp-elf-gdb/bin/xtensa-esp32s3-elf-gdb

zj@a:~$ ls ~/.espressif/tools/
esp-rom-elfs/  esp32ulp-elf/  openocd-esp32/  riscv32-esp-elf/  xtensa-esp-elf/  xtensa-esp-elf-gdb/

zj@a:~/codes/esp32/esp-demo2/myesp$ xtensa-esp32s3-elf-
xtensa-esp32s3-elf-addr2line   xtensa-esp32s3-elf-cc          xtensa-esp32s3-elf-gcc-13.2.0  xtensa-esp32s3-elf-gcov-dump   xtensa-esp32s3-elf-ld.bfd      xtensa-esp32s3-elf-ranlib
xtensa-esp32s3-elf-ar          xtensa-esp32s3-elf-cpp         xtensa-esp32s3-elf-gcc-ar      xtensa-esp32s3-elf-gcov-tool   xtensa-esp32s3-elf-lto-dump    xtensa-esp32s3-elf-readelf
xtensa-esp32s3-elf-as          xtensa-esp32s3-elf-elfedit     xtensa-esp32s3-elf-gcc-nm      xtensa-esp32s3-elf-gdb         xtensa-esp32s3-elf-nm          xtensa-esp32s3-elf-size
xtensa-esp32s3-elf-c++         xtensa-esp32s3-elf-g++         xtensa-esp32s3-elf-gcc-ranlib  xtensa-esp32s3-elf-gprof       xtensa-esp32s3-elf-objcopy     xtensa-esp32s3-elf-strings
xtensa-esp32s3-elf-c++filt     xtensa-esp32s3-elf-gcc         xtensa-esp32s3-elf-gcov        xtensa-esp32s3-elf-ld          xtensa-esp32s3-elf-objdump     xtensa-esp32s3-elf-strip
```

后续每次使用 esp-idf 前需要 `source ~/esp/esp-idf/export.sh(文件不能移动)` 文件：

```shell
# 使用 HTTP 代理而非 SOCKS5 代理, 否则构建时 python 报错: ERROR: Could not install packages due to an OSError: Missing dependencies for SOCKS support.
zj@a:~/codes/esp32/esp-demo$ echo  'export all_proxy="http://192.168.3.2:1080" ALL_PROXY="http://192.168.3.2:1080"' >> ~/esp/esp-idf/export.sh

#alias export_idf='. $HOME/esp/esp-idf/export.sh'
#alias export_esp='. $HOME/esp/export-esp.sh'

zj@a:~/esp/esp-idf$ . ./export.sh
Setting IDF_PATH to '/Users/zhangjun/esp/esp-idf'
Detecting the Python interpreter
Checking "python3" ...
Python 3.12.1
"python3" has been detected
Checking Python compatibility
Checking other ESP-IDF version.
Adding ESP-IDF tools to PATH...
Checking if Python packages are up to date...
Constraint file: /Users/zhangjun/.espressif/espidf.constraints.v5.3.txt
Requirement files:
 - /Users/zhangjun/esp/esp-idf/tools/requirements/requirements.core.txt
Python being checked: /Users/zhangjun/.espressif/python_env/idf5.3_py3.12_env/bin/python
Python requirements are satisfied.
Added the following directories to PATH:
  /Users/zhangjun/esp/esp-idf/components/espcoredump
  /Users/zhangjun/esp/esp-idf/components/partition_table
  /Users/zhangjun/esp/esp-idf/components/app_update
  /Users/zhangjun/.espressif/tools/xtensa-esp-elf-gdb/12.1_20231023/xtensa-esp-elf-gdb/bin
  /Users/zhangjun/.espressif/tools/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin
  /Users/zhangjun/.espressif/tools/riscv32-esp-elf/esp-13.2.0_20230928/riscv32-esp-elf/bin
  /Users/zhangjun/.espressif/tools/esp32ulp-elf/2.35_20220830/esp32ulp-elf/bin
  /Users/zhangjun/.espressif/tools/openocd-esp32/v0.12.0-esp32-20230921/openocd-esp32/bin
  /Users/zhangjun/.espressif/tools/xtensa-esp-elf-gdb/12.1_20231023/xtensa-esp-elf-gdb/bin
  /Users/zhangjun/.espressif/tools/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin
  /Users/zhangjun/.espressif/tools/riscv32-esp-elf/esp-13.2.0_20230928/riscv32-esp-elf/bin
  /Users/zhangjun/.espressif/tools/esp32ulp-elf/2.35_20220830/esp32ulp-elf/bin
  /Users/zhangjun/.espressif/tools/openocd-esp32/v0.12.0-esp32-20230921/openocd-esp32/bin
  /Users/zhangjun/.espressif/python_env/idf5.3_py3.12_env/bin
  /Users/zhangjun/esp/esp-idf/tools
Done! You can now compile ESP-IDF projects.
Go to the project directory and run:

  idf.py build

WARNING: Failed to load shell autocompletion for bash version: 3.2.57(1)-release!
zj@a:~/esp/esp-idf$
```


## <span class="section-num">3</span> esp-idf 开发 {#esp-idf-开发}

ESP-IDF (Espressif IoT Development Framework)  [开发者文档](https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/index.html)：

hello-world 示例：

```shell
zj@a:~$ source ~/esp/esp-idf/export.sh  # 必须 source 安装路径中的 export.sh
zj@a:~$ cd esp/
zj@a:~/esp$ ls
esp-idf/
zj@a:~/esp$ cp -r $IDF_PATH/examples/get-started/hello_world .
zj@a:~/esp$ cd hello_world/

lzj@a:~/esp/hello_world$ ls
CMakeLists.txt  README.md  main/  pytest_hello_world.py  sdkconfig.ci

# 清理项目已经存在的 build 和 configure，生成新的 sdkconfig 文件
zj@a:~/esp/hello_world$ idf.py set-target esp32s3

# 配置项目，结果保存到 sdkconfig 文件
zj@a:~/esp/hello_world$ idf.py menuconfig
Executing action: menuconfig
Running ninja in directory /Users/zhangjun/esp/hello_world/build
Executing "ninja menuconfig"...
。。。
[0/1] cd /Users/zhangjun/esp/hello_world/build && /Users/zhangjun/.espressif/p...F_INIT_VERSION=5.3.0 --output config /Users/zhangjun/esp/hello_world/sdkconfig
TERM environment variable is set to "xterm-256color"
Loaded configuration '/Users/zhangjun/esp/hello_world/sdkconfig'
Configuration (/Users/zhangjun/esp/hello_world/sdkconfig) was not saved

# 构建
zj@a:~/esp/hello_world$ idf.py build
。。。
Creating esp32s3 image...
Merged 2 ELF sections
Successfully created esp32s3 image.
Generated /Users/zhangjun/esp/hello_world/build/bootloader/bootloader.bin
[112/112] cd /Users/zhangjun/esp/hello_world/build/bootloader/esp-idf/esptool_...bootloader 0x0 /Users/zhangjun/esp/hello_world/build/bootloader/bootloader.bin
Bootloader binary size 0x5240 bytes. 0x2dc0 bytes (36%) free.
[980/981] Generating binary image from built executable
esptool.py vv4.8.dev1
Creating esp32s3 image...
Merged 2 ELF sections
Successfully created esp32s3 image.
Generated /Users/zhangjun/esp/hello_world/build/hello_world.bin
[981/981] cd /Users/zhangjun/esp/hello_world/build/esp-idf/esptool_py && /User...able/partition-table.bin /Users/zhangjun/esp/hello_world/build/hello_world.bin
hello_world.bin binary size 0x360c0 bytes. Smallest app partition is 0x100000 bytes. 0xc9f40 bytes (79%) free.

Project build complete. To flash, run:
 idf.py flash
or
 idf.py -p PORT flash
or
 python -m esptool --chip esp32s3 -b 460800 --before default_reset --after hard_reset write_flash --flash_mode dio --flash_size 2MB --flash_freq 80m 0x0 build/bootloader/bootloader.bin 0x8000 build/partition_table/partition-table.bin 0x10000 build/hello_world.bin
or from the "/Users/zhangjun/esp/hello_world/build" directory
 python -m esptool --chip esp32s3 -b 460800 --before default_reset --after hard_reset write_flash "@flash_args"
```

烧写了三个文件:

1.  0x0 build/bootloader/bootloader.bin
2.  0x8000 build/partition_table/partition-table.bin
3.  0x10000 build/hello_world.bin

烧写到设备 Flash, flash 命令自动 build 和 flash：

-   USB 串口：linux: /dev/ttyUSB0, macOS: /dev/cu.

<!--listend-->

```shell
idf.py -p PORT flash

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

idf.py 命令帮助:

```shell
zj@a:~/esp/hello_world$ idf.py --help
Usage: idf.py [OPTIONS] COMMAND1 [ARGS]... [COMMAND2 [ARGS]...]...

  ESP-IDF CLI build management tool. For commands that are not known to idf.py an attempt to execute it as a build system target will be
  made. Selected target: esp32s3

Options:
  --version                       Show IDF version and exit.
  --list-targets                  Print list of supported targets and exit.
  -C, --project-dir PATH          Project directory.
  -B, --build-dir PATH            Build directory.
  -w, --cmake-warn-uninitialized / -n, --no-warnings
                                  Enable CMake uninitialized variable warnings for CMake files inside the project directory. (--no-
                                  warnings is now the default, and doesn't need to be specified.) The default value can be set with the
                                  IDF_CMAKE_WARN_UNINITIALIZED environment variable.
  -v, --verbose                   Verbose build output.
  --preview                       Enable IDF features that are still in preview.
  --ccache / --no-ccache          Use ccache in build. Disabled by default. The default value can be set with the IDF_CCACHE_ENABLE
                                  environment variable.
  -G, --generator [Ninja|Unix Makefiles]
                                  CMake generator.
  --no-hints                      Disable hints on how to resolve errors and logging.
  -D, --define-cache-entry TEXT   Create a cmake cache entry. This option can be used at most once either globally, or for one subcommand.
  -p, --port PATH                 Serial port. The default value can be set with the ESPPORT environment variable. This option can be used
                                  at most once either globally, or for one subcommand.
  -b, --baud INTEGER              Baud rate for flashing. It can imply monitor baud rate as well if it hasn't been defined locally. The
                                  default value can be set with the ESPBAUD environment variable. This option can be used at most once
                                  either globally, or for one subcommand.
  --help                          Show this message and exit.

Commands:
  add-dependency               Add dependency to the manifest file.
  all                          Aliases: build. Build the project.
  app                          Build only the app.
  app-flash                    Flash the app only.
  bootloader                   Build only bootloader.
  bootloader-flash             Flash bootloader only.
  build-system-targets         Print list of build system targets.
  clang-check                  run clang-tidy check under current folder, write the output into "warnings.txt"
  clang-html-report            generate html report to "html_report" folder by reading "warnings.txt" (may take a few minutes). This
                               feature requires extra dependency "codereport". Please install this by running "pip install codereport"
  clean                        Delete build output files from the build directory.
  confserver                   Run JSON configuration server.
  coredump-debug               Create core dump ELF file and run GDB debug session with this file.
  coredump-info                Print crashed task’s registers, callstack, list of available tasks in the system, memory regions and
                               contents of memory stored in core dump (TCBs and stacks)
  create-component             Create a new component.
  create-manifest              Create manifest for specified component.
  create-project               Create a new project.
  create-project-from-example  Create a project from an example.
  delete-version               (Deprecated) Deprecated! New CLI command: "compote component delete". Delete specified version of the
                               component from the component registry.
  dfu                          Build the DFU binary
  dfu-flash                    Flash the DFU binary
  dfu-list                     List DFU capable devices
  docs                         Open web browser with documentation for ESP-IDF
  efuse-common-table           Generate C-source for IDF's eFuse fields.
  efuse-custom-table           Generate C-source for user's eFuse fields.
  encrypted-app-flash          Flash the encrypted app only.
  encrypted-flash              Flash the encrypted project.
  erase-flash                  Erase entire flash chip.
  erase-otadata                Erase otadata partition.
  flash                        Flash the project.
  fullclean                    Delete the entire build directory contents.
  gdb                          Run the GDB.
  gdbgui                       GDB UI in default browser.
  gdbtui                       GDB TUI mode.
  menuconfig                   Run "menuconfig" project configuration tool.
  monitor                      Display serial output.
  openocd                      Run openocd from current path
  pack-component               (Deprecated) Deprecated! New CLI command: "compote component pack". Create component archive and store it
                               in the dist directory.
  partition-table              Build only partition table.
  partition-table-flash        Flash partition table only.
  post-debug                   Utility target to read the output of async debug action and stop them.
  python-clean                 Delete generated Python byte code from the IDF directory
  qemu                         Run QEMU.
  read-otadata                 Read otadata partition.
  reconfigure                  Re-run CMake.
  save-defconfig               Generate a sdkconfig.defaults with options different from the default ones
  set-target                   Set the chip target to build.
  show-efuse-table             Print eFuse table.
  size                         Print basic size information about the app.
  size-components              Print per-component size information.
  size-files                   Print per-source-file size information.
  uf2                          Generate the UF2 binary with all the binaries included
  uf2-app                      Generate an UF2 binary for the application only
  update-dependencies          Update dependencies of the project
  upload-component             (Deprecated) Deprecated! New CLI command: "compote component upload". Upload component to the component
                               registry. If the component doesn't exist in the registry it will be created automatically.
  upload-component-status      (Deprecated) Deprecated! New CLI command: "compote component upload-status". Check the component uploading
                               status by the job ID.
```


### <span class="section-num">3.1</span> build-system {#build-system}


### <span class="section-num">3.2</span> Project Configuration {#project-configuration}

idf.py menuconfig 生成 sdkconfig。

配置参数列表：
<https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-reference/kconfig.html>


### <span class="section-num">3.3</span> JTAG Debugging {#jtag-debugging}

<https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/api-guides/jtag-debugging/index.html>


## <span class="section-num">4</span> 安装 esp rust 开发环境 {#安装-esp-rust-开发环境}

参考：<https://esp-rs.github.io/book/introduction.html>

[安装依赖](https://github.com/esp-rs/esp-idf-template?tab=readme-ov-file#prerequisites):

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

espup 可以同时安装和维护  Xtensa and RISC-V architectures 的工具链。

-   espup 是 rust 开发的工具，取代了 rust-build 项目。
-   Newer Espressif chips are all RISC-V based.

使用 espup 安装 Stensa Rust toolchain 和 custom LLVM 工具链（没有何如 upstream）:

-   不需要手动安装 esp-idf。

<!--listend-->

```shell
# 清理旧版本
zj@a:~/codes/esp32$ espup uninstall
[info]: Uninstalling the Espressif Rust ecosystem
[info]: Uninstallation successfully completed!
zj@a:~/codes/esp32$ rm -rf  ~/.rustup/toolchains/*

# 安装 esp 工具链，必须使用 HTTP 而不能是 SOCKS 代理，否则中间 pip 更新报错
zj@a:~/codes/esp32$ enable_http_proxy

# Install all the necessary tools to develop Rust applications for all supported Espressif target
zj@a:~/codes/esp32$ espup install -l debug
[debug]: connecting to crates.io:443 at 13.32.99.58:443
[debug]: No cached session for DnsName("crates.io")
[debug]: Not resuming any session
[debug]: Using ciphersuite TLS13_AES_128_GCM_SHA256
[debug]: Not resuming
[debug]: TLS1.3 encrypted extensions: [ServerNameAck]
[debug]: ALPN protocol is None
[debug]: created stream: Stream(RustlsStream)
[debug]: sending request GET https://crates.io/api/v1/crates/espup/versions
[debug]: writing prelude: GET /api/v1/crates/espup/versions HTTP/1.1
Host: crates.io
User-Agent: ureq/2.9.1
Accept: */*
accept-encoding: gzip
[debug]: Chunked body in response
[debug]: response 200 to GET https://crates.io/api/v1/crates/espup/versions
[debug]: dropping stream: Stream(RustlsStream)
[info]: Installing the Espressif Rust ecosystem
[debug]: Querying GitHub API: 'https://api.github.com/repos/esp-rs/rust-build/releases/latest'
[debug]: starting new connection: https://api.github.com/
[debug]: Parsing Xtensa Rust version: 1.75.0.0
[debug]: Querying GitHub API: 'https://api.github.com/repos/esp-rs/rust-build/releases'
[debug]: starting new connection: https://api.github.com/
[debug]: Latest Xtensa Rust version: 1.75.0.0
[debug]: Arguments:
            - Export file: "/Users/zhangjun/export-esp.sh"
            - Host triple: x86_64-apple-darwin
            - LLVM Toolchain: Llvm { extended: false, file_name: "libs_llvm-esp-16.0.4-20231113-macos.tar.xz", host_triple: X86_64AppleDarwin, path: "/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp32-elf-clang/esp-16.0.4-20231113", repository_url: "https://github.com/espressif/llvm-project/releases/download/esp-16.0.4-20231113/libs_llvm-esp-16.0.4-20231113-macos.tar.xz", version: "esp-16.0.4-20231113" }
            - Nightly version: "nightly"
            - Rust Toolchain: Some(XtensaRust { cargo_home: "/Users/zhangjun/.cargo", dist_file: "rust-1.75.0.0-x86_64-apple-darwin.tar.xz", dist_url: "https://github.com/esp-rs/rust-build/releases/download/v1.75.0.0/rust-1.75.0.0-x86_64-apple-darwin.tar.xz", host_triple: "x86_64-apple-darwin", path: "/Users/zhangjun/.rustup/toolchains/esp", rustup_home: "/Users/zhangjun/.rustup", src_dist_file: "rust-src-1.75.0.0.tar.xz", src_dist_url: "https://github.com/esp-rs/rust-build/releases/download/v1.75.0.0/rust-src-1.75.0.0.tar.xz", toolchain_destination: "/Users/zhangjun/.rustup/toolchains/esp", version: "1.75.0.0" })
            - Skip version parsing: false
            - Targets: {ESP32C2, ESP32S3, ESP32P4, ESP32C6, ESP32, ESP32S2, ESP32C3, ESP32H2}
            - Toolchain path: "/Users/zhangjun/.rustup/toolchains/esp"
            - Toolchain version: None
[info]: Checking Rust installation
[info]: Installing GCC (xtensa-esp-elf)
[debug]: GCC path: /Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928
[info]: Installing Xtensa Rust 1.75.0.0 toolchain
[info]: Installing Xtensa LLVM
[debug]: Creating directory: '/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp32-elf-clang/esp-16.0.4-20231113'
[info]: Installing RISC-V Rust targets ('riscv32imc-unknown-none-elf', 'riscv32imac-unknown-none-elf' and 'riscv32imafc-unknown-none-elf') for 'nightly' toolchain
[debug]: Creating directory: '/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928'
[info]: Downloading 'rust.tar.xz'
[debug]: starting new connection: https://github.com/
[info]: Downloading 'idf_tool_xtensa_elf_clang.tar.xz'
[debug]: starting new connection: https://github.com/
[info]: Downloading 'xtensa-esp-elf.tar.xz'
[debug]: starting new connection: https://github.com/
[debug]: starting new connection: https://objects.githubusercontent.com/
[debug]: starting new connection: https://objects.githubusercontent.com/
[debug]: starting new connection: https://objects.githubusercontent.com/
[debug]: Extracting tar.xz file to '/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp32-elf-clang/esp-16.0.4-20231113'
[debug]: Extracting tar.xz file to '/Users/zhangjun/.rustup/tmp/.tmpXV6DdJ'
[debug]: Extracting tar.xz file to '/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928'
[info]: Creating symlink between '/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp32-elf-clang/esp-16.0.4-20231113/esp-clang/lib' and '/Users/zhangjun/.espup/esp-clang'
[info]: Installing 'rust' component for Xtensa Rust toolchain
[info]: Downloading 'rust-src.tar.xz'
[debug]: starting new connection: https://github.com/
[debug]: starting new connection: https://objects.githubusercontent.com/
[debug]: Extracting tar.xz file to '/Users/zhangjun/.rustup/tmp/.tmpXV6DdJ'
[info]: Installing 'rust-src' component for Xtensa Rust toolchain
[debug]: Creating export file
[info]: Installation successfully completed!

        To get started, you need to set up some environment variables by running: '. /Users/zhangjun/export-esp.sh'
        This step must be done every time you open a new terminal.
            See other methods for setting the environment in https://esp-rs.github.io/book/installation/riscv-and-xtensa.html#3-set-up-the-environment-variables
```

espup 安装的内容位于 ~/.espup 目录下：

-   Espressif Rust fork with support for Espressif targets
-   nightly toolchain with support for RISC-V targets
-   LLVM fork with support for Xtensa targets
-   GCC toolchain that links the final binary

<!--listend-->

```shell
# 安装了两个 toolchain，分别是 nightly-x86_64-apple-darwin 和 esp
# nightly-x86_64-apple-darwin 是 rust 官方工具链，支持 riscv32xx 和 x86_64-apple-darwin 等 4 个 targets
# 其中 riscv32xx 是 esp32-c3 系列 RISC-V CPU 类型。
zj@a:~/docs$ rustup show
Default host: x86_64-apple-darwin
rustup home:  /Users/zhangjun/.rustup

installed toolchains
--------------------

nightly-x86_64-apple-darwin (default) # 标准工具链
esp  # 自定义 esp 工具链

installed targets for active toolchain
--------------------------------------

riscv32imac-unknown-none-elf  # 标准工具链支持 riscv32 和 x86_64 target
riscv32imafc-unknown-none-elf
riscv32imc-unknown-none-elf
x86_64-apple-darwin

active toolchain
----------------

nightly-x86_64-apple-darwin (default)
rustc 1.78.0-nightly (8ace7ea1f 2024-02-07)


# espup 安装的 esp 工具链, esp 为 channel 名称
ls zj@a:~/docs$ ls -l ~/.rustup/toolchains/esp/
total 0
drwxr-xr-x 13 zhangjun 416  2  8 16:58 bin/   # esp fork 的 rust 交叉编译工具链
drwxr-xr-x  3 zhangjun  96  2  8 14:51 etc/
drwxr-xr-x  6 zhangjun 192  2  8 14:51 lib/
drwxr-xr-x  3 zhangjun  96  2  8 14:50 libexec/
drwxr-xr-x  5 zhangjun 160  2  8 14:51 share/
drwxr-xr-x  3 zhangjun  96  2  8 14:49 xtensa-esp-elf/  # esp fork 的支持 xtensa CPU 的 gcc 等工具链
drwxr-xr-x  3 zhangjun  96  2  8 14:49 xtensa-esp32-elf-clang/ # esp fork 的 clang LLVM 工具链

# esp fork 的 rust 交叉编译工具链
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



zj@a:~/docs$ ls -l  ~/.espup/
total 0
lrwxr-xr-x 1 zhangjun 95  2  8 14:49 esp-clang -> /Users/zhangjun/.rustup/toolchains/esp/xtensa-esp32-elf-clang/esp-16.0.4-20231113/esp-clang/lib/
# ~/esp/export-esp.sh 脚本将 LIBCLANG_PATH 环境便利指向 esp fork 的 LLVM 目录，这样后续 rustc 在编译时自动
# 链接 esp 的版本。（rustc 依赖 LLVM）。
zj@a:~/docs$ ls  ~/.rustup/toolchains/esp/xtensa-esp32-elf-clang/esp-16.0.4-20231113/esp-clang/lib/
clang/  libLLVM.dylib  libclang-cpp.dylib  libclang.dylib


zj@a:~/docs$ ls  ~/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-addr2line*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-ar*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-as*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-c++*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-c++filt*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-cc*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-cpp*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-elfedit*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-g++*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-gcc*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-gcc-13.2.0*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-gcc-ar*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-gcc-nm*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-gcc-ranlib*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-gcov*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-gcov-dump*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-gcov-tool*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-gprof*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-ld*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-ld.bfd*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-lto-dump*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-nm*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-objcopy*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-objdump*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-ranlib*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-readelf*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-size*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-strings*
/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin/xtensa-esp32s3-elf-strip*
zj@a:~/docs$


# export-esp.sh 将该 bin 目录添加 PATH 前面
zj@a:~/docs$ ls   ~/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/xtensa-esp-elf/bin/
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

```rust
zj@a:~/docs$ cat ~/esp/export-esp.sh
export LIBCLANG_PATH="/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp32-elf-clang/esp-16.0.4-20231113/esp-clang/lib"
export PATH="/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928/xtensa-esp-elf/bin:$PATH"
export all_proxy="http://192.168.3.2:1080" ALL_PROXY="http://192.168.3.2:1080"

zj@a:~/docs$ mv ~/export-esp.sh ~/esp/

#alias export_idf='. $HOME/esp/esp-idf/export.sh'
#alias export_esp='. $HOME/esp/export-esp.sh'
```

更新工具链：

```shell
zj@a:~/codes/esp32/esp-demo2/myesp$ enable_socks_proxy
zj@a:~/codes/esp32/esp-demo2/myesp$ espup update
[info]: Updating the Espressif Rust ecosystem
[info]: Checking Rust installation
[warn]: Previous installation of LLVM exists in: '/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp32-elf-clang/esp-16.0.4-20231113'. Reusing this installation
[info]: Installing RISC-V Rust targets ('riscv32imc-unknown-none-elf', 'riscv32imac-unknown-none-elf' and 'riscv32imafc-unknown-none-elf') for 'nightly' toolchain
[info]: Installing GCC (xtensa-esp-elf)
[warn]: Previous installation of GCC exists in: '/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp-elf/esp-13.2.0_20230928'. Reusing this installation
[info]: Creating symlink between '/Users/zhangjun/.rustup/toolchains/esp/xtensa-esp32-elf-clang/esp-16.0.4-20231113/esp-clang/lib' and '/Users/zhangjun/.espup/esp-clang'
[warn]: Previous installation of Xtensa Rust 1.75.0.0 exists in: '/Users/zhangjun/.rustup/toolchains/esp'. Reusing this installation
[info]: Update successfully completed!

        To get started, you need to set up some environment variables by running: '. /Users/zhangjun/export-esp.sh'
        This step must be done every time you open a new terminal.
            See other methods for setting the environment in https://esp-rs.github.io/book/installation/riscv-and-xtensa.html#3-set-up-the-environment-variables
```

esp 是 espup 安装的自定义 toolchain，不能直接安装 rust-analyzer， 需要建一个软链接：

-   <https://www.reddit.com/r/rust/comments/13d2tls/rustanalyzer_and_toolchain_for_esp/>

<!--listend-->

```rust
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

There are the following approaches to using Rust on Espressif chips:

-   Using the std library, a.k.a. `Standard library`.
-   Using `the core library (no_std)`, a.k.a. bare metal development.

In the esp-rs organization, we use the following wording:

-   Repositories starting with `esp-` are focused on `no_std` approach. For example, `esp-hal`
    -   no_std works on top of bare metal, so esp- is an Espressif chip
-   Repositories starting with `esp-idf-` are focused on `std` approach. For example, `esp-idf-hal`
    -   std, apart from bare metal, also needs an additional layer, which is esp-idf-

对于 RISC-V targets，rust 官方是 Tier2 支持，可以直接向官方 rustc 添加对应 target：

```shell
rustup toolchain install nightly --component rust-src

# For no_std (bare-metal) applications
rustup target add riscv32imc-unknown-none-elf # For ESP32-C2 and ESP32-C3
rustup target add riscv32imac-unknown-none-elf # For ESP32-C6 and ESP32-H2

# For std applications
# Since this target is currently Tier 3,
# riscv32imc-esp-espidf 和 riscv32imac-esp-espidf
```

esp32 fork 了 rust 工具链，通过 espup 安装了 esp channel 的 rust 工具链：

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


## <span class="section-num">5</span> esp-idf 和 std 应用 {#esp-idf-和-std-应用}

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


### <span class="section-num">5.1</span> std 应用 {#std-应用}

使用 esp-rs/esp-idf-template 来创建 std 项目

-   cargo generate esp-rs/esp-idf-template cargo：使用 cargo 来构建纯 rust 应用（cargo-frst）；
-   cargo generate esp-rs/esp-idf-template cmake：mix Rust and C/C++ in a traditional ESP-IDF idf.py；

<!--listend-->

```shell
# STD Project
cargo generate esp-rs/esp-idf-template cargo  # 除了 cargo 外，还可以选择 cmake
# NO-STD (Bare-metal) Project
cargo generate esp-rs/esp-template
```

1.  STD cargo-first 项目：
2.  <https://github.com/esp-rs/esp-idf-template/blob/master/README.md>
3.  STD support：When true, adds support for the Rust Standard Library. Otherwise, we will use Rust
    Core Library.
4.  在 .cargo/config.toml 的 env 部分添加：ESP_IDF_TOOLS_INSTALL_DIR = "global"

<!--listend-->

```shell
zj@a:~/codes/esp32/esp-demo$ enable_http_proxy
zj@a:~/codes/esp32/esp-demo2$ cargo generate esp-rs/esp-idf-template cargo
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
zj@a:~/codes/esp32/esp-demo2$ cd myesp


# esp32 rust 项目通过 rust-toolchain.toml 来选择 channel 和 target
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

[unstable]
build-std = ["std", "panic_abort"]

[env]  # 被 embuild 使用的环境变量
MCU="esp32s3"
# Note: this variable is not used by the pio builder (`cargo build --features pio`)
ESP_IDF_VERSION = "v5.2"
# 没有配置 ESP_IDF_TOOLS_INSTALL_DIR = "global" 来使用全局 ~/.espressif/ 工具链，默认是 by 项目 workspace 的。

# 使用 embuild crate 来安装和构建 esp-idf framework
# 对于 non_std 应用，不依赖 esp-idf， 故不需要 build.rs .
cat zj@a:~/codes/esp32/esp-demo2/myesp$ cat build.rs
fn main() {
    embuild::espidf::sysenv::output();
}
# by workspace 安装的 esp-idf 到 .embuild/espressif/ 目录
zj@a:~/codes/esp32/esp-demo2/myesp$ ls -l .embuild/espressif/
total 8.0K
drwxr-xr-x 6 zhangjun  192  2  9 20:36 dist/
drwxr-xr-x 3 zhangjun   96  2  9 20:30 esp-idf/
-rw-r--r-- 1 zhangjun 2.7K  2  9 20:32 espidf.constraints.v5.1.txt
-rw-r--r-- 1 zhangjun 2.8K  2  9 20:27 espidf.constraints.v5.3.txt
drwxr-xr-x 4 zhangjun  128  2  9 20:32 python_env/
drwxr-xr-x 6 zhangjun  192  2  9 20:36 tools/

# dist 下载的内容被解压到 .embuild/espressif/tools/ 目录
zj@a:~/codes/esp32/esp-demo2/myesp$ ls -l .embuild/espressif/dist/
total 151M
-rw-r--r-- 1 zhangjun  70M  2  9 20:35 cmake-3.24.0-macos-universal.tar.gz
-rw-r--r-- 1 zhangjun  16M  2  9 20:36 esp32ulp-elf-2.35_20220830-macos.tar.gz
-rw-r--r-- 1 zhangjun 235K  2  9 20:35 ninja-1.10.2-osx.tar.gz
-rw-r--r-- 1 zhangjun  66M  2  9 20:33 xtensa-esp32s3-elf-12.2.0_20230208-x86_64-apple-darwin.tar.xz  # 交叉编译工具链
zj@a:~/codes/esp32/esp-demo2/myesp$ ls -l .embuild/espressif/tools/
total 0
drwxr-xr-x 3 zhangjun 96  2  9 20:35 cmake/
drwxr-xr-x 3 zhangjun 96  2  9 20:36 esp32ulp-elf/ # ULP (Ultra-Low-Powered)
drwxr-xr-x 3 zhangjun 96  2  9 20:35 ninja/
drwxr-xr-x 3 zhangjun 96  2  9 20:33 xtensa-esp32s3-elf/

# 只包含  xtensa-esp32s3 工具链
zj@a:~/codes/esp32/esp-demo2/myesp$  ls .embuild/espressif/tools/xtensa-esp32s3-elf/esp-12.2.0_20230208/xtensa-esp32s3-elf/bin/
xtensa-esp32s3-elf-addr2line*  xtensa-esp32s3-elf-cc@            xtensa-esp32s3-elf-gcc*         xtensa-esp32s3-elf-gcov*       xtensa-esp32s3-elf-ld.bfd*    xtensa-esp32s3-elf-ranlib*
xtensa-esp32s3-elf-ar*         xtensa-esp32s3-elf-cpp*           xtensa-esp32s3-elf-gcc-12.2.0*  xtensa-esp32s3-elf-gcov-dump*  xtensa-esp32s3-elf-lto-dump*  xtensa-esp32s3-elf-readelf*
xtensa-esp32s3-elf-as*         xtensa-esp32s3-elf-ct-ng.config*  xtensa-esp32s3-elf-gcc-ar*      xtensa-esp32s3-elf-gcov-tool*  xtensa-esp32s3-elf-nm*        xtensa-esp32s3-elf-size*
xtensa-esp32s3-elf-c++*        xtensa-esp32s3-elf-elfedit*       xtensa-esp32s3-elf-gcc-nm*      xtensa-esp32s3-elf-gprof*      xtensa-esp32s3-elf-objcopy*   xtensa-esp32s3-elf-strings*
xtensa-esp32s3-elf-c++filt*    xtensa-esp32s3-elf-g++*           xtensa-esp32s3-elf-gcc-ranlib*  xtensa-esp32s3-elf-ld*         xtensa-esp32s3-elf-objdump*   xtensa-esp32s3-elf-strip*
zj@a:~/codes/esp32/esp-demo2/myesp$

# esp-idf framework，被安装到 python_env/ 目录
zj@a:~/codes/esp32/esp-demo2/myesp$ ls -l .embuild/espressif/esp-idf/v5.1.2/
total 168K
-rw-r--r--  1 zhangjun  11K  2  9 20:30 CMakeLists.txt
-rw-r--r--  1 zhangjun  314  2  9 20:30 CONTRIBUTING.md
-rw-r--r--  1 zhangjun  21K  2  9 20:30 Kconfig
-rw-r--r--  1 zhangjun  12K  2  9 20:30 LICENSE
-rw-r--r--  1 zhangjun 8.3K  2  9 20:30 README.md
-rw-r--r--  1 zhangjun 8.2K  2  9 20:30 README_CN.md
-rw-r--r--  1 zhangjun  475  2  9 20:30 SECURITY.md
-rw-r--r--  1 zhangjun 3.7K  2  9 20:30 SUPPORT_POLICY.md
-rw-r--r--  1 zhangjun 3.4K  2  9 20:30 SUPPORT_POLICY_CN.md
-rw-r--r--  1 zhangjun  721  2  9 20:30 add_path.sh
drwxr-xr-x 79 zhangjun 2.5K  2  9 20:30 components/
-rw-r--r--  1 zhangjun  25K  2  9 20:30 conftest.py
drwxr-xr-x 14 zhangjun  448  2  9 20:30 docs/
drwxr-xr-x 22 zhangjun  704  2  9 20:30 examples/
-rw-r--r--  1 zhangjun 3.9K  2  9 20:30 export.bat
-rw-r--r--  1 zhangjun 3.7K  2  9 20:30 export.fish
-rw-r--r--  1 zhangjun 3.5K  2  9 20:30 export.ps1
-rw-r--r--  1 zhangjun 7.9K  2  9 20:30 export.sh
-rw-r--r--  1 zhangjun 1.6K  2  9 20:30 install.bat
-rwxr-xr-x  1 zhangjun  801  2  9 20:30 install.fish*
-rw-r--r--  1 zhangjun  884  2  9 20:30 install.ps1
-rwxr-xr-x  1 zhangjun  798  2  9 20:30 install.sh*
-rw-r--r--  1 zhangjun  885  2  9 20:30 pytest.ini
-rw-r--r--  1 zhangjun 2.0K  2  9 20:30 sdkconfig.rename
-rw-r--r--  1 zhangjun  568  2  9 20:30 sonar-project.properties
drwxr-xr-x 50 zhangjun 1.6K  2  9 20:32 tools/


zj@a:~/codes/esp32/esp-demo$ source ~/esp/export-esp.sh  # 每次构建前都需要执行
zj@a:~/codes/esp32/esp-demo2/myesp$ cargo build
zj@a:~/codes/esp32/esp-demo$ ls target/
.rustc_info.json       CACHEDIR.TAG           debug/                 xtensa-esp32s3-espidf/
zj@a:~/codes/esp32/esp-demo$ ls -l target/xtensa-esp32s3-espidf/debug/
total 14M
-rw-r--r--   1 zhangjun  21K  2  8 16:04 bootloader.bin   # bootloader
drwxr-xr-x  18 zhangjun  576  2  8 15:59 build/
drwxr-xr-x 178 zhangjun 5.6K  2  8 16:05 deps/
-rwxr-xr-x   1 zhangjun  14M  2  8 16:05 esp-demo*        # 二进制程序
-rw-r--r--   1 zhangjun  177  2  8 16:05 esp-demo.d
drwxr-xr-x   2 zhangjun   64  2  8 15:59 examples/
drwxr-xr-x   3 zhangjun   96  2  8 16:05 incremental/
-rw-r--r--   1 zhangjun 3.0K  2  8 16:03 partition-table.bin  # 分区表
zj@a:~/codes/esp32/esp-demo$ file  target/xtensa-esp32s3-espidf/debug/esp-demo
target/xtensa-esp32s3-espidf/debug/esp-demo: ELF 32-bit LSB executable, Tensilica Xtensa, version 1 (SYSV), statically linked, with debug_info, not stripped
zj@a:~/codes/esp32/esp-demo$
```

1.  STD cmake 项目：
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

对于使用 esp-idf-sys 的 std 应用，项目目录下的 `build.rs` 文件会使用 embuild crate 来下载、编译和链接
esp-idf C framework 和 gcc toolchian。

-   默认是 by 项目下载 esp-idf 的，为了加快速度，可以设置 `ESP_IDF_TOOLS_INSTALL_DIR=global` 来使用

全局 toolchain（~/.espressif/esp-idf/&lt;version&gt;）。

使用 cargo 构建 eps-idf-sys 时， embuild 会读取配置：
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
        -   fromenv - use the build framework from the environment
            -   native builder: use activated esp-idf environment (see esp-idf docs unix / windows)
            -   pio builder: use platformio from the environment (i.e. $PATH)
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
rustc-wrapper = "/Users/zhangjun/.cargo/bin/sccache"

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
ESP_IDF_VERSION = "v5.2"

# 使用全局 ~/.espressif/ 工具链，默认是 by 项目 workspace 的。
ESP_IDF_TOOLS_INSTALL_DIR = "global"
```

sccache 统计信息:

```shell
zj@a:~/codes/esp32/st7735-lcd-examples/esp32c3-examples$ sccache --show-stats
Compile requests                      6
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
Non-cacheable calls                   4
Non-compilation calls                 2
Unsupported compiler calls            0
Average cache write               0.000 s
Average compiler                  0.000 s
Average cache read hit            0.000 s
Failed distributed compilations       0

Non-cacheable reasons:
-                                     2
crate-type                            1
incremental                           1

Cache location                  Local disk: "/Users/zhangjun/Library/Caches/Mozilla.sccache"
Use direct/preprocessor mode?   yes
Version (client)                0.7.7
Max cache size                       10 GiB
zj@a:~/codes/esp32/st7735-lcd-examples/esp32c3-examples$
```


### <span class="section-num">5.2</span> 参考示例 {#参考示例}

1.  [esp-rs/std-trainning](https://github.com/esp-rs/std-training): Embedded Rust Trainings for Espressif
2.  <https://github.com/ivmarkov/rust-esp32-std-demo%EF%BC%9A> Rust on ESP32 STD demo app
3.  <https://apollolabsblog.hashnode.dev/series/esp32-std-embedded-rust%EF%BC%9A> 强烈推荐。
    -   <https://github.com/apollolabsdev/ESP32C3>


## <span class="section-num">6</span> no_std 应用 {#no-std-应用}

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

使用 esp-rs/esp-template 来快速创建 non_std 类型项目：

-   在 .cargo/config.toml 的 env 部分添加 ESP_IDF_TOOLS_INSTALL_DIR = "global"

<!--listend-->

```shell
# STD Project
cargo generate esp-rs/esp-idf-template cargo
# NO-STD (Bare-metal) Project
cargo generate esp-rs/esp-template
```

nonSTD 项目：

-   可以指定是否使用 WiFi/Bluetooth/ESP-NOW via the esp-wifi crate；

<!--listend-->

```nil
zj@a:~/codes/esp32$ enable_http_proxy
zj@a:~/codes/esp32$ cargo generate esp-rs/esp-template
⚠️   Favorite `esp-rs/esp-template` not found in config, using it as a git repository: https://github.com/esp-rs/esp-template.git
🤷   Project Name: esp-dem-nonstd
🔧   Destination: /Users/zhangjun/codes/esp32/esp-dem-nonstd ...
🔧   project-name: esp-dem-nonstd ...
🔧   Generating template ...
✔ 🤷   Which MCU to target? · esp32s3
✔ 🤷   Configure advanced template options? · true
✔ 🤷   Enable WiFi/Bluetooth/ESP-NOW via the esp-wifi crate? · true
✔ 🤷   Enable allocations via the esp-alloc crate? · false
✔ 🤷   Configure project to use Dev Containers (VS Code and GitHub Codespaces)? · false
✔ 🤷   Configure project to support Wokwi simulation with Wokwi VS Code extension? · false
✔ 🤷   Add CI files for GitHub Action? · false
✔ 🤷   Setup logging using the log crate? · false

For more information and examples of esp-wifi showcasing Wifi,BLE and ESP-NOW, see https://github.com/esp-rs/esp-wifi/blob/main/examples.md

🔧   Moving generated files into: `/Users/zhangjun/codes/esp32/esp-dem-nonstd`...
🔧   Initializing a fresh Git repository
✨   Done! New project created /Users/zhangjun/codes/esp32/esp-dem-nonstd
zj@a:~/codes/esp32$

zj@a:~/codes/esp32/esp-dem-nonstd$ cat rust-toolchain.toml
[toolchain]
channel = "esp" # 使用 esp channel 工具链

zj@a:~/codes/esp32/esp-dem-nonstd$ cat .cargo/config.toml
target = "xtensa-esp32s3-none-elf"  # 使用不链接 esp-idf 的 none-elf 工具链

[target.xtensa-esp32s3-none-elf]
runner = "espflash flash --monitor"

[build]
rustflags = [
  "-C", "link-arg=-Tlinkall.x",

  "-C", "link-arg=-Trom_functions.x",

  "-C", "link-arg=-nostartfiles",
]

[unstable]
build-std = ["core"]
zj@a:~/codes/esp32/esp-dem-nonstd$
```


### <span class="section-num">6.1</span> 参考示例 {#参考示例}

1.  <https://apollolabsblog.hashnode.dev/series/esp32c3-embedded-rust-hal> 强烈推荐。
    -   <https://github.com/apollolabsdev/ESP32C3>


## <span class="section-num">7</span> espflash 烧录和监视工具 {#espflash-烧录和监视工具}

espflash 使用 USB 串口(linux: /dev/ttyUSB0, macOS: /dev/cu.) 来烧录芯片:

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

cargo run: 在项目的 .cargo/config.toml 中添加如下内容, 然后就可以执行 argo run 来 flsh 和 monitor 应用:

```rust
[target.'cfg(any(target_arch = "riscv32", target_arch = "xtensa"))']
runner = "espflash flash --baud=921600 --monitor /dev/ttyUSB0"
```

espflash 配置文件 espflash.toml:

1.  Serial port:

<!--listend-->

```toml
[connection]
serial = "/dev/ttyUSB0"
```

1.  baudrate = 460800
2.  bootloader = "path/to/custom/bootloader.bin"
3.  partition_table = "path/to/custom/partition-table.bin"

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


## <span class="section-num">8</span> 报错 {#报错}

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
