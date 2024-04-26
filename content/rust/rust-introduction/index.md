---
title: "Rust 介绍: 安全、性能和生产力"
author: ["于行(张俊，mingduo.zj)", "张俊(zj@opsnull.com)"]
date: 2024-04-26
lastmod: 2024-04-26T17:18:12+08:00
tags: ["rust"]
categories: ["rust"]
draft: false
---

## <span class="section-num">1</span> Rust 初印象 {#rust-初印象}

当前哪个后端编程语言最火？Rust 说自己第二，没人敢说第一。

Rust 连续 8 年霸榜 stackoverflow [最受推崇和喜爱的编程语言](https://survey.stackoverflow.co/2023/#section-admired-and-desired-programming-scripting-and-markup-languages)：

{{< figure src="/images/调研/2024-04-24_11-49-44_screenshot.png" width="400" >}}

推特之父 Jack Dorsey 称 Rust 为 “完美的编程语言”：

{{< figure src="/images/Rust_初印象/2024-04-24_20-03-28_screenshot.png" width="400" >}}

Elon Musk 称 “Rust 是 AGI 时代的编程语言”：

{{< figure src="/images/Rust_初印象/2024-04-24_20-05-36_screenshot.png" width="400" >}}

Microsoft 用 Rust 重写 windows 11 内核：

{{< figure src="/images/Rust_初印象/2024-04-24_20-51-40_screenshot.png" width="400" >}}

Linus 接纳 Rust 称为 Linux 内核的第二编程语言（6.1 开始支持）：

-   Redhat 用 Rust 重写开源 Nividia Nouveau 驱动，新项目名为 [Nova](https://lore.kernel.org/dri-devel/Zfsj0_tb-0-tNrJy@cassiopeiae/?ref=news.itsfoss.com)
-   Google 用 RUst 重写 Android Binder（进程间 IPC 服务），[提升了安全性、可维护性和性能](https://lore.kernel.org/lkml/20231101-rust-binder-v1-0-08ba9197f637@google.com/)；

    {{< figure src="/images/Rust_初印象/2024-04-24_21-07-57_screenshot.png" width="400" >}}

互联网安全研究组(ISRG) 的 [Prossimo 项目](https://www.memorysafety.org/)正在将一些核心开源组件，如 NTP、DNS、TLS 等重写为 Rust 版本，旨在提高内存安全性。

{{< figure src="/images/Rust_初印象/2024-04-25_10-48-14_screenshot.png" width="400" >}}

Cloudflare 用 Rust 实现的 [Pingora 替换 Nginx](https://blog.cloudflare.com/zh-cn/how-we-built-pingora-the-proxy-that-connects-cloudflare-to-the-internet-zh-cn/)：

{{< figure src="/images/Rust_初印象/2024-04-24_20-58-43_screenshot.png" width="600" >}}

Rust 官方嵌入式工作组，芯片厂商 STM32/ESP32 等提供完善的 Rust 开发支持：

-   <https://github.com/esp-rs>  <https://github.com/stm32-rs/stm32-rs>

    {{< figure src="/images/Rust_初印象/2024-04-25_10-52-17_screenshot.png" width="400" >}}


### <span class="section-num">1.1</span> Rust 正吞噬编程世界！ {#rust-正吞噬编程世界}

Rust 之父 Graydon Hoare 是[生物学 nerd](https://www.reddit.com/r/rust/comments/27jvdt/internet_archaeology_the_definitive_endall_source/)，Rust 是以一种健壮、分布式和并行的[真菌命名](https://en.wikipedia.org/wiki/Rust_%28fungus%29)的。

该真菌外形和铁锈类似，正在吞噬系统编程世界！

{{< figure src="/images/Rustacean/2024-04-25_12-09-26_screenshot.png" width="1200" >}}


## <span class="section-num">2</span> 系统编程&amp;语言 {#系统编程-and-语言}

<span class="underline">系统编程</span> ：直接面向操作系统（内核）底层 APIs 和硬件，一般资源受限，对性能、可靠性要求较高。

例如：操作系统内核、驱动程序、嵌入式系统、浏览器、游戏引擎、高并发服务（如 nginx/cache/mq 等）、数据库和中间件、防火墙等。

主要编程语言：C/C++


### <span class="section-num">2.1</span> 为何不是 Java/Go 等语言？ {#为何不是-java-go-等语言}

1.  `性能和效率` ：
    -   C/C++ 提供了接近硬件的编程能力；
    -   Java/Go 需要有资源开销的 Runtime 且 GC 时给应用带来波动的、不可预测的 latency；
2.  `内存和资源管理` ：
    -   C/C++ 允许程序员精细控制内存，这对资源受限或高并发的场景至关重要；
    -   Java/Go 使用垃圾收集机制自动管理内存，增加了预测性能的难度，可能导致不希望的延迟；
3.  `底层控制力` ：
    -   C/C++ 有广泛的系统级 API 支持，适用于操作系统、驱动程序以及直接与硬件交互的其他低级代码；
    -   Java/Go 通过标准库封装，牺牲了程序员对底层细节的控制能力；
4.  `历史和兼容性` ：
    -   许多系统软件（如操内核、嵌入式系统、浏览器等）都是用 C/C++ 编写，使用 C/C++ 更容易集成；
    -   Java/Go 调用这些 C/C++ 代码的库有巨大的性能开销（如 CGO），所以很难实现增量取代他们；

GC 性能波动和 Runtime 开销：

{{< figure src="/images/调研/2024-04-24_12-05-28_screenshot.png" width="500" >}}

![](static/images/调研/2024-04-24_12-06-37_screenshot.png-hidden)
![](static/images/调研/2024-04-24_12-09-00_screenshot.png-hideen)


### <span class="section-num">2.2</span> C/C++ 系统编程困境 {#c-c-plus-plus-系统编程困境}

1.  `内存安全问题` ：
    -   内存管理自由度高，但带来了内存泄露、野指针、缓冲区溢出等问题。导致程序崩溃、数据损坏和漏洞。
    -   C++ 智能指针虽然提供了自动内存管理，但是还是可能有性能、循环引用和错误使用的风险，更加依赖于
        <span class="underline">开发者技术和经验</span> ；
2.  `复杂性和高性能并发编程挑战` ：
    -   对于初学者，指针、内存管理、并发控制等复杂性高，难以调试和修复，需要深厚的技术基础和经验；
    -   Google 开发 Go 语言的初衷：用 C++ 开发、编译大型分布式软件太耗时、太困难。
3.  `对软件工程和生态支持不足` ：
    -   缺少统一的软件包共享、发布机制（系统）：开发者自己下载和管理依赖，保证环境一致性；
    -   开发和调试工具链使用复杂，特别是并发场景支持的不够好；

Microsoft Azure CTO 称新项目应该 <span class="underline">从 C/C++ 转向 Rust</span> ，美国 [NSA](https://media.defense.gov/2022/Nov/10/2003112742/-1/-1/0/CSI_SOFTWARE_MEMORY_SAFETY.PDF)、[Whitehouse](https://www.whitehouse.gov/wp-content/uploads/2024/02/Final-ONCD-Technical-Report.pdf)建议转向 <span class="underline">内存安全语言</span> ：

<div align="center">

<img src="/images/Rust_初印象/2024-04-24_20-56-17_screenshot.png" alt="2024-04-24_20-56-17_screenshot.png" width="600" align="center" /> <img src="/images/Rust_初印象/2024-04-24_20-57-34_screenshot.png" alt="2024-04-24_20-57-34_screenshot.png" width="600" align="center" />

</div>


### <span class="section-num">2.3</span> C/C++ 内存安全问题举例 {#c-c-plus-plus-内存安全问题举例}

Google Chrome 团队发布的 Chrome 被攻击的 Bug 类型分布，内存安全占了一半以上：

{{< figure src="/images/从_C/C++_转向_Rust/2024-04-24_21-22-38_screenshot.png" width="800" >}}

1.  `内存泄漏（Memory Leak）` ：内存泄漏发生在程序分配了内存但未释放，导致内存在程序运行期间不断累积，最终可能耗尽系统资源。例如：
    ```cpp
        int* allocateMemory() {
                int* ptr = new int[10];  // 分配内存
                return ptr;              // 返回指针，但忘记释放内存
        }
        // 这会导致每次调用 allocateMemory 时，分配的内存都不会被释放。
    ```

2.  `缓冲区溢出（Buffer Overflow）` ：当向一个固定大小的缓冲区写入过多数据时，超出的数据会覆盖相邻内存区域。这是常见的安全漏洞来源，可能被用于执行任意代码。例如：
    ```cpp
       void copyData(const char* source) {
           char buffer[10];
           strcpy(buffer, source); // 如果 source 长度超过 10，将发生溢出
       }
    ```

3.  `野指针（Dangling Pointer）` ： 野指针是指向已经释放或无效内存的指针。使用这样的指针可能导致不可预测的行为或程序崩溃。例如：
    ```cpp
       int* ptr = new int(10);
       delete ptr;       // 释放内存
       *ptr = 20;        // 此时 ptr 是野指针，对其解引用是未定义行为
    ```

4.  `双重释放（Double Free）` ：对同一块内存进行两次或多次释放可能导致程序崩溃或其他安全漏洞。例如：
    ```cpp
        char* buffer = new char[100];
        delete[] buffer;  // 第一次释放
        delete[] buffer;  // 第二次释放，可能导致运行时错误
    ```


## <span class="section-num">3</span> Rust 让编程更美好 {#rust-让编程更美好}

Rust 非常适合于追求安全、性能和开发效率的场景，如系统、网络、嵌入式、命令行、WebAssembly 等。

<div align="center">

<img src="/images/Rust_让系统编程更美好/2024-04-24_21-19-01_screenshot.png" alt="2024-04-24_21-19-01_screenshot.png" width="600" align="center" /> <img src="/images/Rust_让编程更美好/2024-04-25_12-41-10_screenshot.png" alt="2024-04-25_12-41-10_screenshot.png" width="600" align="center" />

</div>

| 🎯 | 内存安全   | 可变性、所有权、借用、生命周期，编译通过即承诺 |
|---|--------|-------------------------|
| 🎗️  | 零开销抽象 | 闭包、泛型、迭代器、异步、宏等现代编程范式，性能和手写代码类似 |
| 🎖️  | 无畏并发   | 所有权、可变性、借用检查等特性提前在编译时规避大部分并发编程问题 |
| 🔱 | 异步编程   | 开销更小、并发程度更高的实现              |
| 🔱 | C/C++ 互操作性 | FFI 直接封装和调用存量 C/C++ 库对象和函数 |
| 🙌🏻 | 包管理和工具链 | crate.io 包共享，cargo 包管理和构建工具，rustup 工具链管理等 |
| 🤖 | 编程教练   | 用户友好的编译器出错提示和修复建议        |


## <span class="section-num">4</span> Google Rust 实践经验 {#google-rust-实践经验}

Google Android 工程总监 Lars Bergstrom [在 2024 Rust Nation UK Conference 上称](https://juejin.cn/post/7354974699133190195)：

Google 在将 C++ 代码重写成 Rust 代码时发现：无论是用 Rust 构建服务，还是维护和更新这些用 Rust 编写的服务，所需的工作量都减少了 2 倍以上。并且有 85% 的开发人员对 Rust 代码正确性的信心要高于其他语言。

{{< figure src="/images/Rust_让编程更美好/2024-04-25_10-40-44_screenshot.png" width="800" >}}


## <span class="section-num">5</span> Rust 简史 {#rust-简史}

<div align="center">

<img src="/images/Rust_简史/2024-04-24_20-30-56_screenshot.png" alt="2024-04-24_20-30-56_screenshot.png" width="400" align="center" /> <img src="/images/Rust_简史/2024-04-24_21-35-55_screenshot.png" alt="2024-04-24_21-35-55_screenshot.png" width="400" align="center" />

</div>

-   `2006 诞生` ：Rust 之父格雷登·霍尔（Graydon Hoare）是 Mozilla Research 加拿大程序员。因为电梯软件崩溃，不得不爬 21 层楼梯回到家中，恼火之余想开发一个新语言，能编写小而快速的代码，且不会有内存错误，他将其命名为 Rust。
-   `2010 官宣` ：Mozilla 意识到可以用它构建更好的浏览器引擎 Servo，开始为 Rust 提供财务和法律支持，2010
    年 Mozilla 官宣了该项目和 Rust 语言。
-   `2011 自举` ：Rust编译器初版是由 OCaml 实现，2011 年基于 LLVM 重新实现了编译器并实现了自举。同年，
    Rust 也有了自己的 Logo，其设计灵感来自于[骑自行车的共同爱好-自行车齿盘](https://bugzilla.mozilla.org/show_bug.cgi?id=680521)；
-   `2014 crate.io/cargo` : 前者是 Rust 项目构建管理器，后者则是 Rust 代码的中央包存储库；
-   `2015 1.0 发布` : 1.0 版本发布（比 Go 1.0 晚了 3 年）；
-   `2020 至暗时刻` : Mozilla 解雇了 Servo 引擎团队，包括 Rust 主要贡献者，引起人们对未来的担忧；
-   `2021 Rust 基金会成立` ：五家创始公司（AWS、华为、谷歌、微软和Mozilla）[共同赞助](https://foundation.rust-lang.org/news/2021-02-08-hello-world/)，重新焕发生机和活力；

`当前` ：开放民主、蓬勃发展！

-   基于 RFC（Request For Comments）驱动演进，开放民主。每 6 周发布一个稳定版本（Go 每年 2 个版本）；
-   语言特性不断增强，编译器性能持续优化，生态系统日渐壮大和完善，在系统编程、WebAssembly、嵌入式、大数据、区块链、人工智能等领域蓬勃发展。


## <span class="section-num">6</span> Rust 现状 {#rust-现状}

TIOBE 排名：Rust 热度很高，但排名与 C/C++/Go/Java 还有差距，2024.04 Rust 位列第 19 位，Go 位列第 7
位。

<div align="center">

<img src="/images/Rust_现状/2024-04-25_11-51-49_screenshot.png" alt="2024-04-25_11-51-49_screenshot.png" width="800" align="center" /> <img src="/images/Rust_现状/2024-04-25_11-51-24_screenshot.png" alt="2024-04-25_11-51-24_screenshot.png" width="800" align="center" />

</div>

[Redmonk 2024.3](https://redmonk.com/sogrady/2024/03/08/language-rankings-1-24/) 月排名，Rust 位列 19 位，Go 位列 12 位：

{{< figure src="/images/Rust_现状/2024-04-25_11-57-50_screenshot.png" width="800" >}}

Reddit 社区活跃度（2024.04.25 北京时间 12:00 ）：Rust 社区活跃度要高于 Go

<div align="center">

<img src="/images/Rust_现状/2024-04-25_12-02-26_screenshot.png" alt="2024-04-25_12-02-26_screenshot.png" width="600" align="center" /> <img src="/images/Rust_现状/2024-04-25_12-02-43_screenshot.png" alt="2024-04-25_12-02-43_screenshot.png" width="600" align="center" />

</div>


### <span class="section-num">6.1</span> 用户采纳 {#用户采纳}

1.  AWS：Rust 在 AWS 内部基础设施构建上发挥的[关键作用](https://aws.amazon.com/cn/blogs/opensource/how-our-aws-rust-team-will-contribute-to-rusts-future-successes/)，[Firecracker](https://firecracker-microvm.github.io/)，[Bottlerocket](https://aws.amazon.com/cn/bottlerocket/)，[Nitro System](https://aws.amazon.com/cn/blogs/aws/aws-nitro-enclaves-isolated-ec2-environments-to-process-confidential-data/)
2.  Huawei：正在努力将部分代码库迁移到 Rust，[可信编程实验室](https://trusted-programming.github.io/)
3.  Google：已将 Rust 应用到 Chromium、[Android](https://security.googleblog.com/2022/12/memory-safe-languages-in-android-13.html) 和 FuchsiaOS 中
4.  Microsoft：正在用 Rust [重写 Windows 11](https://www.theregister.com/2023/04/27/microsoft_windows_rust/) 代码
5.  Meta：唯一非创始成员的铂金赞助商；[A brief history of Rust at Facebook](https://engineering.fb.com/2021/04/29/developer-tools/rust/)
6.  OpenAI：2024 发布的 grok-1 大模型中，Rust 开发的 [Qdrant 向量数据库](https://github.com/xai-org/qdrant)也发挥了重要作用，也是 Rust 在
    AI 领域应用迈出的重要一步。
7.  Cloudflare：开源了其内部替代 nginx 的 Rust 库 [pingora](https://github.com/cloudflare/pingora)，作为业界一家提供互联网基础设施和网络服务的公司，其采用 Rust 的示范效应也是非常明显的。
8.  indfluxdb：3.0 版本用 Rust 重写。很多其他新兴数据库也都在用 Rust 重写，如 greptimedb、cnosdb、
    CeresDB 等；
9.  Linux 基金会：Linux Kernel 6.1 版本[对 Rust 提供了支持](https://www.infoq.com/news/2022/12/linux-6-1-rust/)。
    -   Rust 同时进入 Windows、Linux 内核，江湖地位进一步提升！

国内实践：

1.  蚂蚁：KCL 项目 [性能提升 40 倍！我们用 Rust 重写了自己的项目](https://ata.atatech.org/articles/12000256636?spm=a1z2e.12184483.0.0.23804f9bhWn5CF)
2.  蚂蚁：[星绽（Asterinas）](https://github.com/asterinas/asterinas)，一个基于Rust语言的下一代OS内核，兼容Linux，但比Linux更安全可靠。
3.  PingCap：tikv 时序数据库；
4.  字节跳动: 开源了诸如 Rust RPC 框架 volo、基于 io-uring 的 Rust async runtime monoio 等。


### <span class="section-num">6.2</span> Rustacean {#rustacean}

和 Go 开发人员自称 Gopher 类似，Rust 开发人员自称 Rustacean，这是一个结合了 Rust 和 Crustacean（甲壳类）两个词语的组合词。

社区非官方吉祥物(mascot)：[Ferris，一只可爱的红色螃蟹](https://www.rustacean.net/)。它象征着 Rust 语言的安全性、并发性和生产力，同时也代表着 Rust 社区的活跃和友好。

{{< figure src="/images/Rustacean/2024-04-25_12-14-52_screenshot.png" width="900" >}}


## <span class="section-num">7</span> Rust 价值观 {#rust-价值观}

Rust 核心价值观：Rust 核心组成员 Stephen Klabnik 在 QCon London 发表了一次名为 [How Rust Views
Tradeoffs](https://www.infoq.com/presentations/rust-tradeoffs/) 的演讲，这些价值观是 Rust team 在做设计取舍时拒绝妥协的点，包括： <span class="underline">内存安全 &gt; 执行速度 &gt;
生产力</span> ：

这三个价值观是 Rust 语言的设计目标，是 Rust 语言的特色和优势所在，是 Rust 核心团队判断语言演进方向的根本依据。

<div align="center">

<img src="/images/Rust_设计哲学/2024-04-25_12-45-26_screenshot.png" alt="2024-04-25_12-45-26_screenshot.png" width="600" align="center" /> <img src="/images/Rust_设计哲学/2024-04-25_12-47-56_screenshot.png" alt="2024-04-25_12-47-56_screenshot.png" width="600" align="center" />

</div>


### <span class="section-num">7.1</span> Rust vs Go 价值观 {#rust-vs-go-价值观}

{{< figure src="/images/Rust_设计哲学/2024-04-25_13-42-30_screenshot.png" width="800" >}}

Go 官方介绍隐含的 Go 设计哲学和价值观：

1.  `简单（Easy）` ：Simple 是 Go 的首要设计原则，开发者可以快速上手形成生产力。Go 去掉了很多其他语言中的复杂特性，如类型层次、继承等；
2.  `弹性（Scalable）` ：体现在语言内置并发、大规模软件工程协同等方面。goroutine 轻量级并发，面向接口抽象，基于 Module 的可重现编译、构建，极高的编译速度、高质量标准库、实用工具链等。
3.  `鲁棒（Robust）` ：Go 通过 GC 来自动管理内存，通过 goroutine 和 channel 实现并发编程，避免数据竞争。同时提供了显式的错误处理机制。

总结：

-   Rust 更注重安全、底层控制和极致性能；
-   Go 则更加关注简单、安全、扩展性与工程效率。
-   两者在定位和设计哲学上存在区别，但也有一些共同特点，比如都拥有现代的工具链、活跃的社区等。


### <span class="section-num">7.2</span> 内存安全 {#内存安全}

编程语言的内存管理分一般分为两种：

1.  自己显式申请和释放，如 C (malloc/free) 和 C++ (new/delete)
2.  语言本身无需关心，由运行时进行垃圾回收(Java, Go)

而 Rust 采用了第三种方式，通过语言层面的强制规则，在编译时最大程度地检查出这些错误，从而保证程序的内存安全。

1.  `对象可变性` ：
    -   默认不可变，需要用 mut 关键字来明确指示对象可变；
    -   **收益** ：语言层面确保只读的数据不能被篡改；
2.  `对象所有权` ：
    -   任何对象都只有一个变量 Owner，它拥有该对象，在对象离开作用域时销毁它；
    -   所有权可以被转移，这时原来 Owner 变量是未初始化状态，不能再使用；
    -   **收益** ：不需要手动释放对象，对象也不可能被多次释放 《-- <span class="underline">避免内存泄露和竞态条件</span>
3.  `借用检查` ：
    -   借用：类似于 C/C++ 指针，是在不获得对象所有权的情况下使用对象；
    -   借用可变性：不可变借用（ `&T` ）、可变借用（ `&mut T` ）；
    -   借用检查：对象可以有多个不可变借用，但只能有一个可变借用。对象有借用时，不能转移所有权和修改。
    -   **收益** ：支持对象共享访问，但同时只能有一个可变借用来修改对象；《-- <span class="underline">无畏并发！</span>
4.  `生命周期检查` ：
    -   任何对象都有生命周期，一般是块级作用域。
    -   生命周期检查：当借用一个对象时，借用变量的生命周期不能长于借用对象： `let b = &T;`
    -   **收益** ：编译器确保借用指针不会指向无效对象；
5.  `强类型系统` ：
    -   任何对象都具有类型，不同类型之间需要转换；
    -   **收益** ：避免了类型错误和缓冲区溢出；

由于 Rust 能够在编译时就检查出内存错误，开发者就不必再花费大量时间和精力去寻找和修复这些错误了。Rust
的内存安全机制不仅能够提高程序的稳定性和可靠性，还能够降低开发和维护的难度。

<!--list-separator-->

1.  举例：客户信息管理

    ```rust
    use std::collections::HashMap;

    // Customer 拥有 name/email 的所有权。
    struct Customer {
        id: u32,
        name: String,
        email: String,
    }

    impl Customer {
        fn new(id: u32, name: String, email: String) -> Self {
            Customer { id, name, email }
        }
    }

    struct Logger {}

    impl Logger {
        fn log(&self, message: &str) {
            println!("Log: {}", message);
        }
    }

    // CustomerManager 拥有其内部 HashMap 的所有权。
    //
    // 生命周期注解 'a 指定了 CustomerManager 实例中的 logger 字段引用的生命周期。这意味着任何
    // CustomerManager 实例都必须确保它引用的 Logger 对象在 CustomerManager 存在期间不被丢弃。
    //
    // 这种方式能确保 Logger 的引用始终有效，避免悬挂引用的问题，同时演示了如何在实际应用中使用生命周期
    // 来保证数据引用的安全性。
    struct CustomerManager<'a> {
        customers: HashMap<u32, Customer>,
        next_id: u32,
        logger: &'a Logger,  // 对 Logger 的引用，使用生命周期注解
    }

    impl<'a> CustomerManager<'a> {
        fn new(logger: &'a Logger) -> Self {
            CustomerManager {
                customers: HashMap::new(),
                next_id: 1,
                logger,
            }
        }

        // 当更新或删除客户信息时，通过 &mut self 参数来借用管理器的可变引用，确保在方法调用期间管理器的
        // 数据结构不被其他所有者错误修改。
        fn add_customer(&mut self, name: String, email: String) -> u32 {
            let id = self.next_id;
            self.customers.insert(id, Customer::new(id, name.clone(), email));
            self.logger.log(&format!("Added customer {} with ID {}", name, id));
            self.next_id += 1;
            id
        }

        // 错误处理：在 remove_customer 和 update_customer 方法中，使用 Result 类型处理可能的错误，避免
        // 了异常抛出带来的不确定性和安全隐患。
        fn remove_customer(&mut self, id: u32) -> Result<(), String> {
            if self.customers.remove(&id).is_some() {
                self.logger.log(&format!("Removed customer with ID {}", id));
                Ok(())
            } else {
                Err(format!("No customer with id {}", id))
            }
        }

        // 迭代打印 customer 信息
        fn list_customers(&self) {
            for customer in self.customers.values() {
                println!("ID: {}, Name: {}, Email: {}", customer.id, customer.name, customer.email);
            }
        }
    }

    fn main() {
        let logger = Logger {};
        // 借用 logger
        let mut manager = CustomerManager::new(&logger);
        // manager 是可变的，可以调用方法修改它
        manager.add_customer("Alice Smith".to_string(), "alice@example.com".to_string());
        manager.add_customer("Bob Johnson".to_string(), "bob@example.com".to_string());

        manager.remove_customer(1).unwrap();
        manager.list_customers();

        // // 错误案例 1: 生命周期问题
        // let mut manager2;
        // {
        //     let temp_logger = Logger {};
        //     manager2 = CustomerManager::new(&temp_logger); // temp_logger 不会活得比 manager2 更久
        //     manager2.add_customer("Temporary Customer".to_string(), "temp@example.com".to_string());
        // } // temp_logger 在这里被销毁，manager2 中的 logger 引用变为悬挂引用
        // manager2.list_customers();

        // // 错误案例 2: 可变和不可变借用冲突
        // let mutable_borrow = &mut manager;
        // let immuta`ble_borrow = &manager;
        // mutable_borrow.add_customer("Charlie Brown".to_string(), "charlie@example.com".to_string());
        // // 这里违反了同时存在可变借用和不可变借用的规则
        // immutable_borrow.list_customers();

        // // 错误案例 3: 可变性错误
        // let  immutable_manager = CustomerManager::new(&logger);
        // // 试图在存在不可变借用的情况下进行修改
        // immutable_manager.add_customer("Diana Prince".to_string(), "diana@example.com".to_string());
    }
    ```


### <span class="section-num">7.3</span> 高性能 {#高性能}

根据 [benchmark-game](https://benchmarksgame-team.pages.debian.net/benchmarksgame/performance/binarytrees.html) 的评测，Rust 语言速度仅次于 C/C++，远高于其他语言。

<div align="center">

<img src="/images/高性能/2024-04-25_14-11-25_screenshot.png" alt="2024-04-25_14-11-25_screenshot.png" width="600" align="center" /> <img src="/images/高性能/2024-04-25_14-11-57_screenshot.png" alt="2024-04-25_14-11-57_screenshot.png" width="600" align="center" />

</div>

`Rust vs C vs Go 性能测试` ：运行一个计算 Fibonacci 数列第 40 项的任务，该任务是计算密集型，考验各语言递归解法的性能：

C 版本：

```C
#include <stdio.h>
#include <time.h>

unsigned long long fibonacci(unsigned int n) {
    if (n == 0 || n == 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

int main() {
    clock_t start = clock();
    unsigned long long result = fibonacci(40);
    clock_t end = clock();
    double time_taken = (double)(end - start) / CLOCKS_PER_SEC;
    printf("Result: %llu, Time taken: %f ms\n", result, time_taken * 1000);
    return 0;
}
```

Rust 版本：

```rust
use std::time::Instant;

fn fibonacci(n: u64) -> u64 {
    if n == 0 || n == 1 {
        return n;
    }
    fibonacci(n - 1) + fibonacci(n - 2)
}

fn main() {
    let start = Instant::now();
    let result = fibonacci(40);
    let duration = start.elapsed();
    println!("Result: {}, Time taken: {:?}", result, duration);
}
```

Go 版本：

```go
package main

import (
    "fmt"
    "time"
)

func fibonacci(n uint) uint {
    if n == 0 || n == 1 {
        return n
    }
    return fibonacci(n-1) + fibonacci(n-2)
}

func main() {
    start := time.Now()
    result := fibonacci(40)
    duration := time.Since(start)
    fmt.Printf("Result: %d, Time taken: %v\n", result, duration)
}
```

<!--list-separator-->

1.  Rust 高性能秘诀

    -   `零成本抽象` ：Rust 提供的高级抽象，如迭代器、闭包等，在编译时都会被优化到与手写底层代码相同的性能。
        -   <span class="underline">"高级语句别怕慢，编译之后都一样"</span>

    -   `无垃圾收集` ：通过所有权和借用，无 GC 开销，内存管理更精确，性能表现可预测。
        -   Rust 对象有唯一的所有权，可以在传参、返回、赋值时转移，避免数据赋值。

    -   `较少的运行时` ：几乎没有运行时开销。除了必要的堆栈展开和任务（如在使用 async 时）外，它没有垃圾收集器，也没有其他重的运行时特性。
        -   Rust 程序的启动时间和运行效率都非常高。

    -   `LLVM 架构编译优化` ：充分利用语言和 LLVM 架构提供的高级编译时优化技术。
        -   Rust 默认不可变性有助于避免状态改变引起的错误，通过确定某些资源在其生命周期内不会被修改，从而实现更有效的资源管理和访问。

<!--list-separator-->

2.  无畏并发

    Rust 旨在成为一种充分利用现代多核处理器能力的高性能系统编程语言，并发是 Rust 语言设计的一部分。

    -   C/C++/Jave 语言层面没有直接支持并发编程，而是使用线程库或并发库来实现。

    Rust 将并发编程中常见的死锁问题、竞态问题提前暴露到编译时刻，从而做到 <span class="underline">无畏并发</span> ：

    1.  `所有权和借用检查` ：编译器通过所有权和生命周期规则强制每个数据只能有一个可写引用或多个只读引用，这在编译时就阻止了数据竞争的可能性。

    2.  `消息传递` ： Rust 倡导使用消息传递来进行线程间的通信，避免共享状态和加锁；

    3.  `同步和通信原语` ：Rust 标准库提供了多种同步原语，如互斥锁（Mutex）、条件变量（Condvar）、信号量（Semaphore）和原子操作（Atomic类型），这些都是构建多线程应用程序的基础。

    4.  `异步编程` ：Rust Future/async/await 语法提供了异步编程支持，允许开发者以近似同步的方式写异步代码。

    5.  `无锁编程` ：Rust 通过 Atomic 类型和其他无锁数据结构支持无锁编程，使得高性能的并发程序更容易实现。

    并发编程举例：并发生成和打印数据向量，演示独占锁 Mutex、通道 mpsc、线程 thread、无锁编程 AtomicUsize

    ```rust
    use std::sync::{Arc, Mutex, mpsc};
    use std::thread;
    use std::sync::atomic::{AtomicUsize, Ordering};

    fn main() {
        // 创建一个通道
        let (tx, rx) = mpsc::channel();

        // 使用 Arc 和 Mutex 保护一个共享向量
        let shared_data = Arc::new(Mutex::new(vec![]));

        // 使用 Arc 和 AtomicUsize 跟踪完成的操作数
        let operations_count = Arc::new(AtomicUsize::new(0));

        // 创建多个生产者线程
        const NUM_THREADS: usize = 5;
        let mut handles = vec![];

        for i in 0..NUM_THREADS {
            let tx_clone = tx.clone();
            let data_clone = Arc::clone(&shared_data);
            let count_clone = Arc::clone(&operations_count);

            let handle = thread::spawn( move || {
                let data = i * 10; // 每个线程生成特定的数据

                // 将数据添加到共享向量中
                {
                    let mut data_vec = data_clone.lock().unwrap();
                    data_vec.push(data);
                }

                // 原子地更新操作计数器
                count_clone.fetch_add(1, Ordering::SeqCst);

                // 发送数据到通道
                tx_clone.send(data).unwrap();
            });

            handles.push(handle);
        }

        // 数据处理线程
        let handle_receiver = thread::spawn(move || {
            let mut data_vec = vec![];
            for _ in 0..NUM_THREADS {
                let received = rx.recv().unwrap();
                data_vec.push(received);
            }
            println!("Received data: {:?}", data_vec);
        });

        // 等待所有生产者线程结束
        for handle in handles {
            handle.join().unwrap();
        }

        // 等待接收线程结束
        handle_receiver.join().unwrap();

        // 打印 Mutex 保护的向量内容和原子计数器的值
        let data_vec = shared_data.lock().unwrap();
        println!("Final vector state: {:?}", *data_vec);
        println!("Total operations completed: {}", operations_count.load(Ordering::SeqCst));
    }
    ```

<!--list-separator-->

3.  异步编程

    <span class="underline">同步编程模型</span> ：使用阻塞式 IO，每个进入的连接通常需要一个单独的线程或进程来处理。（如 Apache HTTP）

    1.  连接数限制：线程栈内存开销导致系统不能创建大量的线程；
    2.  性能下降：内核创建和切换线程时开销大，导致内核分配给应用的时间片减少；

    <span class="underline">异步编程模型</span> ：使用非阻塞式 IO，通过事件驱动和 IO 多路复用技术，用很少的线程来处理大量的并发：

    1.  `非阻塞 I/O` ： 非阻塞套接字、非阻塞文件操作、信号（SIGIO）等，提供事件驱动机制；
    2.  `I/O 多路复用` ：poll/select/epoll/io_uring;
        -   **poll:** 每次调用 poll 都需要遍历整个文件描述符列表，大量文件描述符的情况下，性能可能会变得很差。
        -   **select:** 它有一个固定的文件描述符限制，并且性能不如其他机制。
        -   **epoll:** 可以处理大量的文件描述符而不会有性能问题，并且支持边缘触发。但不支持文件 I/O；
        -   **io_uring:** 同时支持套接字和文件 IO，可以避免用户空间和内核空间之间的数据拷贝。
    3.  `事件循环` ：管理所有非阻塞 I/O 的事件，并对执行读写回调操作。

    异步编程模型是当今主流的高并发编程模型，在 Node.js/Nginx/Go 等项目中得到广泛应用。

<!--list-separator-->

4.  异步编程举例

    C 语言版本：异步并发 TCP 服务器。

    ```C
    #include <stdio.h>
    #include <stdlib.h>
    #include <string.h>
    #include <unistd.h>
    #include <arpa/inet.h>
    #include <sys/socket.h>
    #include <sys/epoll.h>

    #define MAX_EVENTS 10
    #define BUFFER_SIZE 1024

    int main() {
        int server_fd, client_fd, epoll_fd, event_count, i;
        struct sockaddr_in server_addr, client_addr;
        struct epoll_event event, events[MAX_EVENTS];
        char buffer[BUFFER_SIZE];

        // Create server socket
        if ((server_fd = socket(AF_INET, SOCK_STREAM, 0)) == -1) {
            perror("socket");
            exit(EXIT_FAILURE);
        }

        // Set server socket options
        int opt = 1;
        if (setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)) == -1) {
            perror("setsockopt");
            exit(EXIT_FAILURE);
        }

        // Bind server socket
        memset(&server_addr, 0, sizeof(server_addr));
        server_addr.sin_family = AF_INET;
        server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
        server_addr.sin_port = htons(8080);
        if (bind(server_fd, (struct sockaddr *)&server_addr, sizeof(server_addr)) == -1) {
            perror("bind");
            exit(EXIT_FAILURE);
        }

        // Listen for incoming connections
        if (listen(server_fd, SOMAXCONN) == -1) {
            perror("listen");
            exit(EXIT_FAILURE);
        }

        // Create epoll instance
        if ((epoll_fd = epoll_create1(0)) == -1) {
            perror("epoll_create1");
            exit(EXIT_FAILURE);
        }

        // Add server socket to epoll
        event.events = EPOLLIN;
        event.data.fd = server_fd;
        if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, server_fd, &event) == -1) {
            perror("epoll_ctl");
            exit(EXIT_FAILURE);
        }

        // Event loop
        while (1) {
            event_count = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
            for (i = 0; i < event_count; ++i) {
                if (events[i].data.fd == server_fd) {
                    // Accept new client connection
                    socklen_t client_addr_len = sizeof(client_addr);
                    client_fd = accept(server_fd, (struct sockaddr *)&client_addr, &client_addr_len);
                    if (client_fd == -1) {
                        perror("accept");
                        exit(EXIT_FAILURE);
                    }
                    // Set client socket to non-blocking
                    int flags = fcntl(client_fd, F_GETFL, 0);
                    fcntl(client_fd, F_SETFL, flags | O_NONBLOCK);
                    // Add client socket to epoll
                    event.events = EPOLLIN | EPOLLET;
                    event.data.fd = client_fd;
                    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, client_fd, &event) == -1) {
                        perror("epoll_ctl");
                        exit(EXIT_FAILURE);
                    }
                } else {
                    // Read data from client socket
                    int bytes_read = read(events[i].data.fd, buffer, BUFFER_SIZE);
                    if (bytes_read == -1) {
                        if (errno != EAGAIN && errno != EWOULDBLOCK) {
                            perror("read");
                            exit(EXIT_FAILURE);
                        }
                    } else if (bytes_read == 0) {
                        // Client closed connection
                        close(events[i].data.fd);
                        epoll_ctl(epoll_fd, EPOLL_CTL_DEL, events[i].data.fd, NULL);
                    } else {
                        // Echo data back to client
                        write(events[i].data.fd, buffer, bytes_read);
                    }
                }
            }
        }

        return 0;
    }
    ```

    用 Rust 重写：代码更简洁（行数 104 -&gt; 33 行），不需要处理底层复杂的非阻塞 I/O 设置、事件循环、线程创建和同步等，开发效率更高！

    ```rust
    //! ```cargo
    //! [dependencies]
    //!  tokio = { version = "1", features = ["full"] }
    //! ```
    use std::error::Error;
    use tokio::net::{TcpListener, TcpStream};
    use tokio::io::{AsyncReadExt, AsyncWriteExt};

    #[tokio::main]
    async fn main() -> Result<(), Box<dyn Error>> {
        let listener = TcpListener::bind("127.0.0.1:8080").await?;
        println!("Server listening on port 8080");

        loop {
            let (mut socket, _) = listener.accept().await?;
            tokio::spawn(async move {
                let mut buf = vec![0; 1024];
                loop {
                    match socket.read(&mut buf).await {
                        // Socket closed
                        Ok(0) => break,
                        Ok(n) => {
                            // Echo received data back to client
                            if let Err(e) = socket.write_all(&buf[..n]).await {
                                eprintln!("Failed to write to socket: {}", e);
                                break;
                            }
                        }
                        Err(e) => {
                            eprintln!("Failed to read from socket: {}", e);
                            break;
                        }
                    }
                }
            });
        }
    }
    ```

    Rust 异步编程的核心是 Future 和 async/await：

    1.  Future 表示一个异步操作的结果，而 async/await 则是一种语法糖，可以方便地编写异步代码，使其看起来像是同步的。
    2.  当一个函数被标记为 async 时，它将返回一个 Future，该 Future 可以在后台执行，并在完成时产生结果。

    举例：echo server/client

    ```rust
    // Echo Server
    use tokio::net::TcpListener;
    use tokio::io::{AsyncReadExt, AsyncWriteExt};

    #[tokio::main]
    async fn main() {
        // 绑定监听端口
        let listener = TcpListener::bind("127.0.0.1:8080").await.unwrap();

        println!("Server running on 127.0.0.1:8080");

        loop {
            // 接收连接
            let (mut socket, _) = listener.accept().await.unwrap();

            tokio::spawn(async move {
                let mut buf = vec![0; 1024];

                // 循环读取数据并回显
                loop {
                    let n = match socket.read(&mut buf).await {
                        Ok(n) if n == 0 => return, // 无数据可读，关闭连接
                        Ok(n) => n,
                        Err(e) => {
                            eprintln!("Failed to read from socket; err = {:?}", e);
                            return;
                        }
                    };

                    // 发送回客户端
                    if let Err(e) = socket.write_all(&buf[0..n]).await {
                        eprintln!("Failed to write to socket; err = {:?}", e);
                        return;
                    }
                }
            });
        }
    }


    // Echo Client
    use tokio::net::TcpStream;
    use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
    use tokio::select;

    #[tokio::main]
    async fn main() {
        // 连接到服务器
        let mut stream = TcpStream::connect("127.0.0.1:8080").await.unwrap();
        println!("Connected to the server!");

        let mut stdin = io::stdin(); // 获取标准输入句柄
        let mut buffer = vec![0; 1024];

        loop {
            let bytes_read = stdin.read(&mut buffer).await.unwrap();

            if bytes_read == 0 {
                break; // 没有更多输入
            }

            // 发送数据到服务器
            stream.write_all(&buffer[..bytes_read]).await.unwrap();

            // 清空缓冲区以接收响应
            buffer.clear();
            buffer.resize(1024, 0);

            // 等待响应
            let bytes_read = stream.read(&mut buffer).await.unwrap();
            println!("Received: {}", String::from_utf8_lossy(&buffer[..bytes_read]));
        }
    }
    ```


### <span class="section-num">7.4</span> 生产力 {#生产力}

生产力是 Rust 的第三个核心价值观，通过各种工具和系统让开发者能够更轻松地编写、调试和维护 Rust 程序。

| 工具          | 功能                                    |
|-------------|---------------------------------------|
| rustup        | Rust 工具链管理（安装、升级等）         |
| crates.io     | 官方的 Rust 包共享、发现                |
| cargo         | crate 包下载、Rust 程序的 build/test/bench 等工具 |
| docs.rs       | Rust 包在线文档                         |
| clippy        | Rust 程序静态 linter                    |
| rust-analyzer | Rust 的 LSP 语言服务器，为各种编辑器提供智能编辑能力 |

<div align="center">

<img src="/images/生产力/2024-04-25_15-41-43_screenshot.png" alt="2024-04-25_15-41-43_screenshot.png" width="550" align="center" /> <img src="/images/生产力/2024-04-25_15-44-28_screenshot.png" alt="2024-04-25_15-44-28_screenshot.png" width="550" align="center" /> <img src="/images/生产力/2024-04-25_15-45-13_screenshot.png" alt="2024-04-25_15-45-13_screenshot.png" width="550" align="center" />

</div>


## <span class="section-num">8</span> Rust 特性示例 {#rust-特性示例}


### <span class="section-num">8.1</span> Hello Rust! {#hello-rust}

这是一个简单的 "Hello, Rust 💥!" 例子。

```rust
// 这是一个简单的 "hello Rust!" 例子。

fn main() {
    let 中文 = "asdfadsf";
      println!("Hello, Rust 💥! {:?} {} {} ", "adfdf".to_string(), 3, 中文)
}
```


### <span class="section-num">8.2</span> 内置类型 {#内置类型}

基本类型（如整型、浮点型、布尔型、字符型等）和一些更复杂的扩展类型（如元组、数组、结构体、枚举等）

```rust
#![allow(dead_code, unused_variables)]

fn main() {
    // 1. 基本类型
    let boolean: bool = true;
    let integer: i32 = 42;
    let float: f32 = 3.14;
    let character: char = 'R';
    // 类型推导
    let string = "Hello, World!";
    // 格式化输出
    println!("#{:20.20}#", string);


    // 2. 元组
    let tuple: (i32, f64, char) = (500, 6.4, 'x');
    println!("Tuple: ({}, {}, {})", tuple.0, tuple.1, tuple.2);


    // 3. 数组
    let array: [i32; 3] = [1, 2, 3];
    println!("Array: [{}, {}, {}]", array[0], array[1], array[2]);
    //array[0] = 2; // error[E0594]: cannot assign to `array[_]`, as `array` is not declared as mutable

    // 可变数组，类型推导
    let mut values = [10, 20, 30];
    values[0] = 11;

    // 切片
    let slice = &values[..];
    println!("slice: {:?}", slice); // Debug 输出

    // 迭代数组
    for (i,v) in values.iter().enumerate() {
        println!("value[{}]: {}", i, v);
    }


    // 4. 结构体
    // C 风格
    #[derive(Debug)]
    struct Person {
        name: String,
        age: u8,
    }
    // 元组风格
    struct Pair(i32, f32);
    // 无任何 field
    struct Unit;

    // 为 Person 添加一个方法
    impl Person {
        fn set_name(&mut self, name: String) {
            self.name = name;
        }
    }

    // 为 Person 实现一个 trait
    impl std::fmt::Disply for Person{
        fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result{
            write!(f, "{}: {}", self.name, self.age)
        }
    }

    // 创建结构体对象
    let name = "zhangjun".to_string();
    let person = Person{name, age: 18 };
    let person2 = Person{name: "mingduo.zj".to_string(), ..person}; // 对象展开
    println!("person: ({:?}, {:?})", person, person2);
    // 从已有 person 创建新的可变 person
    let mut person = Person{..person2}; // 同名变量遮盖
    person.set_name("zhangjun".to_string());
    println!("person: {}", person);

    // 解构  结构体对象
    let pair = Pair(1.1, 1);
    let Pair(integer: f64, decimal:f64 ) = pair;
    println!("Pair: ({}, {})", integer, decimal);


    // 5. 枚举
    #[derive(Debug)]
    enum WebEvent {
        // An `enum` variant may either be `unit-like`,
        PageLoad,
        PageUnload,
        // like tuple structs,
        KeyPress(char),
        Paste(String),
        // or c-like structures.
        Click { x: i64, y: i64 },
    }

    // 创建枚举对象
    let pressed = WebEvent::KeyPress('x');
    let pasted  = WebEvent::Paste("my text".to_owned());
    let click   = WebEvent::Click { x: 20, y: 80 };
    let load    = WebEvent::PageLoad;
    let unload  = WebEvent::PageUnload;
    // 模式匹配
    match click {
        WebEvent::PageLoad => println!("page loaded"),
        WebEvent::PageUnload => println!("page unloaded"),
        WebEvent::KeyPress(c) => println!("pressed '{}'.", c),
        WebEvent::Paste(s) => println!("pasted \"{}\".", s),
        WebEvent::Click { x, y: 60 } => {
            println!("clicked at x={}.", x);
        },
        _ => {
            println!("WebEvent: {:?}", click);
        }
    }
}
```


### <span class="section-num">8.3</span> 集合类型 {#集合类型}

Vec（动态数组）、HashMap（键值对集合）、HashSet（唯一元素集合）、以及更传统的数组

```rust
#![allow(dead_code, unused_variables)]

use std::collections::{HashMap, HashSet};

fn main() {
    // 1. Vec: 动态数组，可增长的列表
    let mut vec = Vec::new();
    vec.push(1);
    vec.push(2);
    vec.push(3);
    println!("Vec: {:?}", vec);
    // 使用宏快速创建数组
    let vec = vec![1, 2, 3];
    // 切片和借用
    let int_slice = &vec[..];
    // 多次共享借用
    let int_slice: &[i32] = &vec;

    // 2. 固定数组: 固定大小的集合
    let array = [4, 5, 6];
    println!("Array: {:?}", array);


    // 3. HashMap: 键值对集合
    let mut map = HashMap::new();
    map.insert("color", "red");
    map.insert("size", "10");
    map.insert("shape", "circle");
    println!("HashMap: {:?}", map);
    // entry 不存在时创建
    map.entry("entry").or_insert("value");
    // 迭代
    for (k, v) in map {
        println!("\t k: {}, v: {}", k, v);
    }

    // 4. HashSet: 唯一元素的集合，自动去重
    let mut set = HashSet::new();
    set.insert("apple");
    set.insert("banana");
    set.insert("cherry");
    set.insert("apple"); // 这个 "apple" 不会被加入，因为已经存在
    println!("HashSet: {:?}", set);
}
```


### <span class="section-num">8.4</span> 迭代器和闭包 {#迭代器和闭包}

迭代器是一种高效遍历、处理集合元素的方式，也可实现迭代器 trait 来为自定义类型添加迭代支持。

闭包是一种匿名函数，一般短小精悍，可以捕获上下文中的对象，在迭代器、并发等场景广泛应用。

```rust
fn main() {
    // 创建一个向量
    let nums = vec![1, 2, 3, 4, 5];

    // 使用迭代器和闭包遍历并打印每个元素
    nums.iter().for_each(|&x| println!("Number: {}", x));

    // 使用 map 方法转换每个元素
    let squares: Vec<i32> = nums.iter().map(|&x| x * x).collect();
    println!("Squares: {:?}", squares);

    // 使用 filter 方法筛选元素
    let evens: Vec<&i32> = nums.iter().filter(|x| x % 2 == 0).collect();
    println!("Even numbers: {:?}", evens);

    // 使用 fold 方法累加元素
    let sum: i32 = nums.iter().fold(0, |acc, &x| acc + x);
    println!("Sum: {}", sum);

    // 使用 zip 组合两个迭代器
    let names = vec!["Alice", "Bob", "Carol"];
    let ages = vec![28, 25, 32];
    let people: Vec<_> = names.iter().zip(ages.iter()).collect();
    println!("People: {:?}", people);

    // 使用 enumerate 获取带索引的迭代器
    for (index, value) in nums.iter().enumerate() {
        println!("Index: {}, Value: {}", index, value);
    }

    // 使用 find 查找第一个匹配的元素
    if let Some(&first_even) = nums.iter().find(|&&x| x % 2 == 0) {
        println!("First even number: {}", first_even);
    }

    // 使用 take_while 取连续满足条件的部分
    let initial_positives: Vec<i32> = nums.iter().take_while(|&&x| x > 0).cloned().collect();
    println!("Initial positives: {:?}", initial_positives);
}
```

迭代器非常适合于大量数据的处理，比如：使用 Rayon 提供的并行迭代器来加速数据处理过程：

-   Rayon 通过工作窃取算法来优化任务分配，使得并行运算非常高效。

<!--listend-->

```rust
//! ```cargo
//! [dependencies]
//!  rayon = { version = "1.10.0"}
//! ```
extern crate rayon;
use rayon::prelude::*;

fn main() {
    // 创建一个较大的向量
    let nums: Vec<i64> = (1..=10_000_000).collect();

    // 使用 Rayon 的并行迭代器来计算所有数字的平方和
    let now = std::time::Instant::now();
    let sum_of_squares: i64 = nums.par_iter() // 使用并行迭代器
        .map(|&x| x * x)
        .sum(); // 并行计算总和
    println!("rayon: {}, time: {}", sum_of_squares, now.elapsed().as_millis());

    // 使用默认迭代器计算
    let now = std::time::Instant::now();
    let sum_of_squares: i64 = nums.iter()
        .map(|&x| x * x)
        .sum();
    println!("non-rayon: {}, time: {}", sum_of_squares, now.elapsed().as_millis());
}
```

为自定义类型实现迭代器：

```rust
// 定义一个斐波那契数列生成器的结构体
struct Fibonacci {
    curr: u64,  // 当前数
    next: u64,  // 下一个数
}

// 为 `Fibonacci` 实现方法
impl Fibonacci {
    // 构造函数，创建一个新的斐波那契生成器
    fn new() -> Self {
        Fibonacci { curr: 0, next: 1 }
    }
}

// 为 `Fibonacci` 实现 `Iterator` 特征
impl Iterator for Fibonacci {
    type Item = u64;

    // 定义如何生成下一个元素
    fn next(&mut self) -> Option<Self::Item> {
        let new_next = self.curr + self.next; // 计算新的下一个值
        self.curr = self.next;               // 更新当前值
        self.next = new_next;                // 更新下一个值

        Some(self.curr)                      // 返回当前值
    }
}

fn main() {
    // 创建一个新的斐波那契数列生成器
    let fib = Fibonacci::new();

    // 打印前 10 个斐波那契数
    for number in fib.take(10) {
        println!("{}", number);
    }
}
```


### <span class="section-num">8.5</span> 对象可变性、借用和生命周期 {#对象可变性-借用和生命周期}

Rust 中的对象可变性、借用和生命周期是 Rust 的核心特性之一，它们共同确保了内存安全性和数据竞争的防止，而无需垃圾回收。

```rust
#![allow(unused_assignments, unused_variables)]

fn main() {
    let z = 10; // 不可变对象
    //z = 9; // error[E0384]: cannot assign twice to immutable variable `z`

    let r = &z; // 不可变引用
    // *r = 9; // error[E0594]: cannot assign to `*r`, which is behind a `&` reference
    println!("z: {}, accessed via reference r: {}", z, r); // 同时使用不可变引用和原始数据


    let mut x = 5; // 可变变量定义
    x = 6; // 可以修改 x
    {
        let y = &mut x; // 可变引用
        *y += 1;
    } // y 的生命周期在这里结束，x 的可变借用结束
    println!("x: {}", x); // 此时可以安全地访问 x

    let y1 = &x;
    let y2 = &mut x; // error[E0502]: cannot borrow `x` as mutable because it is also borrowed as immutable
    let y3 = &mut x; // error[E0499]: cannot borrow `x` as mutable more than once at a time
    println!("{} {} {}", y1, y2, y3);
    println!("{} {}", y2, y3);

    let string1 = String::from("Rust");
    let result;
    {
        let string2 = String::from("Programming");
        result = longest(&string1, &string2);
    }
    // let string2 = String::from("Programming");
    // result = longest(&string1, &string2);
    println!("The longest string is {}", result);
}

// 'a 是一个生命周期参数
fn longest<'a, 'b: 'a>(x: &'a str, y: &'b str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```


### <span class="section-num">8.6</span> 模式匹配 {#模式匹配}

模式匹配是 Rust 的一个核心特性，它允许开发者以非常表达性和安全的方式检查和解构值。

```rust
#![allow(dead_code)]

// 定义一个简单的枚举来表示 HTTP 状态码
enum HttpStatus {
    Ok,
    NotFound,
    Unauthorized,
    Unknown(u16), // 未知状态码
}

// 定义一个结构体表示一个简单的点
struct Point {
    x: i32,
    y: i32,
}

// 定义一个元组结构体
struct Color(u8, u8, u8);

// 函数，返回一个元组结构体 Color
fn get_color() -> Color {
    Color(255, 0, 0) // 红色
}

fn main() {
    // 使用 if let 来简化枚举处理
    let status = HttpStatus::Unauthorized;
    if let HttpStatus::Unauthorized = status {
        println!("Access denied!");
    }

    // 使用 while let 来处理循环中的模式匹配
    let mut numbers = vec![Some(3), None, Some(2), Some(1)];
    while let Some(num) = numbers.pop() {
        println!("Popped number: {:?}", num);
    }

    // 解构元组
    let tuple = (3, "hello", 4.5);
    match tuple {
        (3, _, f) if f > 4.0 => println!("The integer is three, and the float is greater than 4.0"),
        (_, s, _) => println!("The string is '{}'", s),
    }

    // 函数传参和返回值使用模式匹配
    let Color(r, g, b) = get_color();
    println!("Color - Red: {}, Green: {}, Blue: {}", r, g, b);
}
```


### <span class="section-num">8.7</span> 错误处理 {#错误处理}

在 Rust 中，错误处理是通过使用 Result 和 Option 类型以及相关的方法来完成的。这种方法避免了异常处理机制的使用，强调显式错误检查和处理，从而提高代码的可靠性和安全性

```rust
#![allow(dead_code)]

use std::fmt;
use std::error::Error;

#[derive(Debug)]
enum MyError {
    Io(std::io::Error),
    Parse(std::num::ParseIntError),
    Custom(String),
}

impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match *self {
            MyError::Io(ref err) => write!(f, "IO error: {}", err),
            MyError::Parse(ref err) => write!(f, "Parse error: {}", err),
            MyError::Custom(ref desc) => write!(f, "Custom error: {}", desc),
        }
    }
}

impl Error for MyError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match *self {
            MyError::Io(ref err) => Some(err),
            MyError::Parse(ref err) => Some(err),
            MyError::Custom(_) => None,
        }
    }
}

impl From<std::io::Error> for MyError {
    fn from(err: std::io::Error) -> MyError {
        MyError::Io(err)
    }
}

impl From<std::num::ParseIntError> for MyError {
    fn from(err: std::num::ParseIntError) -> MyError {
        MyError::Parse(err)
    }
}

use std::fs::File;
use std::io::prelude::*;

fn read_file_to_string(path: &str) -> Result<String, MyError> {
    let mut file = File::open(path)?;
    let mut content = String::new();
    file.read_to_string(&mut content)?;
    Ok(content)
}

fn parse_content_to_number(content: String) -> Result<i32, MyError> {
    let trimmed = content.trim();
    let num: i32 = trimmed.parse()?;
    Ok(num)
}

fn process_file(path: &str) -> Result<i32, MyError> {
    let content = read_file_to_string(path)?;
    let number = parse_content_to_number(content)?;
    Ok(number)
}

fn main() {
    match process_file("numbers.txt") {
        Ok(num) => println!("Number read: {}", num),
        Err(e) => println!("Error: {}", e),
    }
}
```


### <span class="section-num">8.8</span> 宏编程 {#宏编程}

宏是 Rust 提供的高级特性，可以用来控制编译器行为，如条件编译、编译器警告和错误、测试和基准测试。

宏本质上是 Rust 编译器扩展，可以用来实现代码生成、DSL 等特性。

```rust
//! ```cargo
//! [dependencies]
//!  serde = { version = "1.0", features = ["derive"] }
//!  serde_json = "1.0"
//! ```
extern crate serde;
extern crate serde_json;
use serde::{Serialize, Deserialize};

// 条件编译
#[cfg(target_os = "linux")]
fn are_you_on_linux() {
    println!("You are running linux!");
}

#[cfg(not(target_os = "linux"))]
fn are_you_on_linux() {
    println!("You are not running linux!");
}

// 序列号和反序列化
#[derive(Serialize, Deserialize, Debug)]
struct Person {
    name: String,
    age: u32,
    is_student: bool,
}

fn main() {
    are_you_on_linux();

    // 创建一个 Person 对象
    let person = Person {
        name: "John Doe".to_string(),
        age: 30,
        is_student: false,
    };

    // 序列化 Person 对象为 JSON 字符串
    let serialized = serde_json::to_string(&person).unwrap();
    println!("Serialized Person: {}", serialized);

    // 反序列化 JSON 字符串回 Person 对象
    let deserialized: Person = serde_json::from_str(&serialized).unwrap();
    println!("Deserialized Person: {:?}", deserialized);
}
```


### <span class="section-num">8.9</span> 文件、命令和 HTTP {#文件-命令和-http}

Rust 有完善的标准库，为日常编程场景提供丰富的支持：

```rust
//! ```cargo
//! [dependencies]
//! reqwest = { version = "0.11", features = ["blocking"] }
//! serde = { version = "1.0", features = ["derive"] }
//! serde_json = "1.0"
//! ```
use std::fs::{self, File};
use std::io::{Write, Read};
use std::process::Command;
use reqwest::blocking::{Client, Response};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 文件读写
    let file_path = "example.txt";
    let content_to_write = "Hello, Rust!";

    // 写入文件
    let mut file = File::create(file_path)?;
    writeln!(file, "{}", content_to_write)?;

    // 读取文件
    let mut file = File::open(file_path)?;
    let mut content = String::new();
    file.read_to_string(&mut content)?;
    println!("File content: {}", content);

    // 创建目录
    let dir_path = "example_dir";
    fs::create_dir_all(dir_path)?;

    // 执行命令并获取输出
    let output = Command::new("echo")
        .arg("Hello from command line")
        .output()?;
    println!("Command output: {}", String::from_utf8_lossy(&output.stdout));

    // HTTP GET 请求
    let res = http_get("https://httpbin.org/get")?;
    println!("GET request response: {}", res);

    // HTTP POST 请求
    let res = http_post("https://httpbin.org/post", "{\"name\":\"Rust\"}")?;
    println!("POST request response: {}", res);

    Ok(())
}

// 发送 HTTP GET 请求的函数
fn http_get(url: &str) -> Result<String, reqwest::Error> {
    let client = Client::new();
    let res: Response = client.get(url).send()?;
    let body = res.text()?;
    Ok(body)
}

// 发送 HTTP POST 请求的函数
fn http_post(url: &str, body: &str) -> Result<String, reqwest::Error> {
    let client = Client::new();
    let res: Response = client.post(url)
                              .header("Content-Type", "application/json")
                              .body(body.to_string())
                              .send()?;
    let body = res.text()?;
    Ok(body)
}
```


### <span class="section-num">8.10</span> 测试 {#测试}

Rust 中测试是语言的一部分，Rust 提供了内置的测试支持，使得编写单元测试、集成测试和性能测试（基准测试）变得非常简单。

单元测试，src/lib.rs：

```rust
pub fn add_two(a: i32) -> i32 {
    a + 2
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_adds_two() {
        assert_eq!(4, add_two(2));
        //assert_eq!(3, add_two(2));
    }
}

fn main() {}
```

集成测试: tests/integration_test.rs：

```rust
// 引入库，这里的库名假设为 `my_crate`
extern crate my_crate;

#[test]
fn test_add_two() {
    assert_eq!(5, my_crate::add_two(3));
}

// 性能测试：基准测试，benches/benchmar.rs
#![feature(test)]

extern crate test;

#[cfg(test)]
mod tests {
    extern crate test;
    use test::Bencher;
    use super::super::*;

    #[bench]
    fn bench_add_two(b: &mut Bencher) {
        b.iter(|| add_two(2));
    }
}
```


### <span class="section-num">8.11</span> 包和模块 {#包和模块}

模块系统和 Cargo 的包管理功能支持大型项目的构建和依赖管理。模块系统帮助组织代码结构，而 Cargo 管理依赖、构建过程和包版本，非常适合现代软件开发的需要。

Rust 项目目录解构：

```text
my_project/
├── Cargo.toml
├── src/
│   ├── lib.rs              # 主库文件
│   ├── another_lib.rs      # 第二个库文件
│   ├── main.rs             # 默认的二进制文件入口
│   └── bin/
│       ├── binary1.rs      # 第一个额外的二进制文件
│       └── binary2.rs      # 第二个额外的二进制文件
├── benches/
│   ├── bench1.rs           # 第一个基准测试文件
│   └── bench2.rs           # 第二个基准测试文件
└── tests/
    ├── test_lib.rs         # 对主库的测试
    └── test_another_lib.rs # 对第二个库的测试
```

Cargo.toml 中定义了项目的基本信息，依赖管理、特征（features）配置、工作空间（workspace）设置和一些常用的元数据。

```toml
[package]
name = "example_project"              # 项目名称
version = "0.1.0"                     # 项目版本
edition = "2018"                      # 使用的 Rust 版本，常见的是2018或2021
authors = ["Your Name <you@example.com>"]  # 作者信息
description = "An example Rust project"    # 项目描述
license = "MIT"                       # 许可证类型
repository = "https://github.com/username/example_project"  # 项目仓库URL
readme = "README.md"                  # README 文件位置
homepage = "https://example.com"      # 项目主页
documentation = "https://docs.example.com" # 项目文档链接

# 编译器（rustc）的额外参数
rustc-flags = ["-A warnings"]         # 忽略所有警告

[dependencies]
serde = { version = "1.0", features = ["derive"] } # 带特征的依赖
log = "0.4.8"                         # 指定版本号的依赖
reqwest = { version = "0.11", default-features = false } # 禁用默认特征的依赖
tokio = { version = "1", features = ["full"] }     # 启用特定特征的依赖

[dev-dependencies]
tokio-test = "1.0"                    # 只在开发和测试中使用的依赖
async-std = "1.9"                     # 用于异步编程的开发依赖

[build-dependencies]
cc = "1.0"                            # 构建脚本使用的依赖

[features]
default = ["use-serde", "http2"]       # 默认特征
use-serde = ["serde/derive"]          # 可选特征
http2 = []                            # 空特征组

[workspace]
members = ["member1", "member2"]      # 包含的工作空间成员

[profile.release]
opt-level = 3                         # 发布模式的优化级别
lto = true                            # 启用链接时优化
debug = false                         # 禁用调试信息

[profile.dev]
opt-level = 0                         # 开发模式的优化级别
debug = true                          # 启用调试信息

# 自定义二进制文件输出路径
[package.metadata.cargo-make]
workspace = false
```

通过将代码风格到独立的 crate 和 module 文件中，可以灵活组织大型复杂的代码：

```rust
// 在 src/lib.rs 中定义模块

// pub 控制模块和其中内容的 可见性。
pub mod math {
    pub fn add(a: i32, b: i32) -> i32 {
        a + b
    }
}

// 在其他文件中使用模块
use my_crate::math;

fn main() {
    let result = math::add(1, 2);
    println!("1 + 2 = {}", result);
}
```


### <span class="section-num">8.12</span> 异步编程 {#异步编程}

异步编程模型支持高效的并发编程，适用于IO密集型任务。这是通过 async 和 await 关键字及强大的生态系统（如 Tokio）来实现的。

```rust
//! ```cargo
//! [dependencies]
//!  tokio = { version = "1", features = ["full"] }
//!  reqwest = "0.12.4"
//! ```
async fn fetch_data() -> Result<(), reqwest::Error> {
    let response = reqwest::get("https://example.com").await?;
    println!("Response: {}", response.status());
    Ok(())
}

#[tokio::main]
async fn main() {
    match fetch_data().await {
        Ok(()) => println!("Success"),
        Err(e) => println!("Error: {}", e),
    }
}
```


### <span class="section-num">8.13</span> 多种编程范式 {#多种编程范式}

-   面向对象编程：通过定义 Animal trait 和它的实现类 Dog 和 Cat 展示了多态和封装。
-   泛型编程：count_sounds 函数使用泛型来对不同类型的 Animal 进行操作，展示了如何写出通用的函数。
-   函数式编程：使用 .iter() 和 .filter() 来统计符合条件的元素数量，体现了函数式编程的风格。
-   命令式编程：在 main 函数和其他地方使用了常规的循环和条件语句。
-   宏编程：通过 spawn_threads! 宏简化了多线程的创建过程。
-   并发编程：通过 Arc 和 Mutex 来共享和修改数据，确保线程安全。

<!--listend-->

```rust
use std::sync::{Arc, Mutex};
use std::thread;

// 定义一个 Animal trait，展示面向对象编程（多态和封装）
trait Animal {
    fn make_sound(&self) -> &'static str;
}

// Dog 结构体实现 Animal trait
struct Dog;
impl Animal for Dog {
    fn make_sound(&self) -> &'static str {
        "Woof"
    }
}

// Cat 结构体实现 Animal trait
struct Cat;
impl Animal for Cat {
    fn make_sound(&self) -> &'static str {
        "Meow"
    }
}

// 定义一个泛型函数，展示泛型编程
fn count_sounds<T: Animal + ?Sized>(animals: &[&T], sound: &str) -> usize {
    animals.iter().filter(|a| a.make_sound() == sound).count()
}

// 定义一个宏来简化线程创建
macro_rules! spawn_threads {
    ($data:expr, $processor:expr) => {
        {
            let mut handles = vec![];
            for _ in 0..4 {  // 创建四个线程
                let data = $data.clone();
                let handle = thread::spawn(move || {
                    $processor(data);
                });
                handles.push(handle);
            }
            handles
        }
    };
}

fn main() {
    // 创建动物列表
    let animals: Vec<&dyn Animal> = vec![&Dog, &Cat, &Dog];

    // 使用函数式编程方式统计声音
    let woof_count = count_sounds(&animals, "Woof");
    let meow_count = count_sounds(&animals, "Meow");
    println!("Woof count: {}", woof_count);
    println!("Meow count: {}", meow_count);

    // 创建共享状态以进行并发处理
    let counter = Arc::new(Mutex::new(0));
    let handles = spawn_threads!(counter, |counter: Arc<Mutex<usize>>| {
        let mut num = counter.lock().unwrap();
        *num += 1;  // 递增共享计数器
    });

    // 等待所有线程完成
    for handle in handles {
        handle.join().unwrap();
    }

    println!("Counter: {}", *counter.lock().unwrap());
}
```


### <span class="section-num">8.14</span> 外部函数接口（Foreign Function Interface, FFI） {#外部函数接口-foreign-function-interface-ffi}

通过 FFI，Rust 可以直接调用大量的现有 C/C++ 库且没有开销，这是 Rust 在系统编程领域极具竞争力的优势。

```rust
// 1. 创建 C 函数文件
#include <stdint.h>

int32_t add(int32_t a, int32_t b) {
    return a + b;
}

// 2. 编译 C 库
gcc -shared -fPIC -o libadd.so add.c

// 3. 编写 Rust 代码
extern crate libc;
use libc::int32_t;

// 声明外部函数
extern "C" {
    fn add(a: int32_t, b: int32_t) -> int32_t;
}

fn main() {
    let x = 5;
    let y = 10;
    unsafe {
        // 调用 C 函数
        let result = add(x, y);
        println!("Result of adding {} and {}: {}", x, y, result);
    }
}

// 4. 编译、链接和执行
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/path/to/your/library
cargo run
```


## <span class="section-num">9</span> 感受&amp;总结 {#感受-and-总结}

1.  Rust 是通用、现代、实用的语言，语法特性非常具有表现力，使用场景涵盖了硬件、内核、系统到网络、服务、命令行等丰富领域，在安全&amp;性能方面也是一骑绝尘；
    -   值得关注和投资，特别是想突破一下自己的领域局限，又不想学很多语言，可以考虑下 Rust；

2.  Rust 程序需要经过良好思考和设计，只追求短平快结果的程序员会觉得 Rust 开发效率极低！
    -   Rust 有助于创建良好设计、更可靠，长期来看更低维护成本的代码，引领程序员整体水平的提升；

3.  Rust 的学习曲线比较陡峭，编译器非常严格，初学者绝大部分时间都在和编译错误做斗争！
    -   需要付出大量时间精力来学习语言、标准库、最佳编程范式和常用三方库。
    -   可以把 ChatGPT 作为 Rust 私人导师，学习、解惑、Debug 无所不能！


## <span class="section-num">10</span> 参考 {#参考}

1.  [深受程序员喜爱的Rust（上）](https://grow.alibaba-inc.com/course/4800014498471831/section/1800014504682131?spm=a1z24uy2.26994894.0.0.55696ebaz1e5Re)
2.  [深受程序员喜爱的Rust（下）](https://grow.alibaba-inc.com/course/4800014498471831/section/1800014498482031?spm=a1z24uy2.26994894.0.0.55696ebaz1e5Re)
3.  [为什么要学一学 Rust？](https://mp.weixin.qq.com/s?__biz=MzkwMTMwNDE4Mw==&mid=2247484236&idx=1&sn=2207f010dfd4b077ab7bb0123705ef9d&chksm=c0b79954f7c01042f62156b7260eb625960f6bc8696cbcad10c01656760a4ee31d2141d9fdf6&scene=21#wechat_redirect)
4.  [Gopher的Rust第一课：Rust的那些事儿](https://tonybai.com/2024/04/22/gopher-rust-first-lesson-all-about-rust/)
