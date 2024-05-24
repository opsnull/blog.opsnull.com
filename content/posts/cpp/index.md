---
title: "C 预处理器-个人参考手册"
author: ["张俊(zj@opsnull.com)"]
date: 2024-05-24T00:00:00+08:00
lastmod: 2024-05-24T17:35:58+08:00
tags: ["C", "cpp", "Tools"]
categories: ["C"]
draft: false
---

这是我个人的 C 预处理器参考手册文档。

<!--more-->

<details>
<summary>变更历史</summary>
<div class="details">

2024-05-24 Fri  首次创建
</div>
</details>

C 预处理器，简称 cpp，是一个宏处理器语言和工具。


## <span class="section-num">1</span> 头文件内容 {#头文件内容}

头文件一般包含声明（extern 变量、常量和函数签名声明，struct/union/enum 类型的前向声明）、宏定义（常量宏、带参数的函数宏）、typedef 类型定义等，不能包含常量定义、变量定义、函数定义，它们需要在 c 源文件中定义。

C 不允许变量定义、常量定义、函数定义、struct/union/enum 重复定义，否则编译报错， `所以这些定义一般在
C 源文件而非头文件中定义（因为头文件会被不同的源文件包含，可能导致重复定义）` ，如果要使用这些唯一的定义：

-   对于 struct/union/enum 可以使用前向声明方式来使用；
-   对于变量、常量、函数，可以通过 extern 声明的方式来使用；

头文件中建议包含的内容：

1.  **宏定义** ：使用 #define 指令定义常量或宏函数，可以重复定义， `但如果定义不一致会告警` 。只对本文件有效(编译器参数 -D 定义的宏对所有源文件生效，如 gcc -DBUFFER_SIZE=1024 file1.c file2.c -o myprogram）
    -   \#define 宏的作用域从其定义处开始，直到文件末尾或遇到 #undef 指令为止。如果要在多个源文件中共享，需要在一个头文件中定义。
        ```C
        // 宏定义不以分号结尾
        #define PI 3.14159
        #define MAX(a,b) ((a) > (b) ? (a) : (b))

        // 宏定义也可以不包含值
        #define EXTRA_HAPPY
        #ifdef EXTRA_HAPPY
        //...
        #endif
        ```

2.  **typedef 类型定义** ：使用 `typedef` 指令定义新的类型名称，可以重复定义，但是定义必须完全一致，否则
    error：
    -   文件级别（全局作用域）：如果 typedef 定义在任何函数之外，则它在定义所在的整个文件中有效，并且通过包含相应的头文件，可以在其他文件中使用。
    -   块级别（局部作用域）：如果 typedef 定义在一个函数内部，它的作用域仅限于该函数内部。
        ```C
        typedef unsigned int uint;
        typedef struct {
                int x;
                int y;
        } Point;
        ```

3.  **函数原型声明** ：声明函数原型，可以重复声明，但是需要确保函数签名完全一致：
    ```C
    void printHello(void);
    int add(int a, int b);
    ```

4.  **外部变量声明** ：使用 \`extern\` 关键字声明在其他文件中定义的全局变量，可以重复声明，但是类型必须要一致：

    ```text
    extern int globalVar;
    ```

5.  **内联函数** ：定义内联函数（通常是小型函数），以便在头文件中直接实现。
    ```C
        inline int square(int x) {
            return x * x;
        }
    ```

6.  **预处理器条件编译指令** ：如 \`#ifdef\`、\`#ifndef\`、\`#endif\` 等，用于条件编译。
    ```C
    #ifndef MY_HEADER_H
    #define MY_HEADER_H
    // 内容
    #endif

    #ifdef EXTRA_HAPPY
        printf("I'm extra happy!\n");
    #endif

    #ifndef EXTRA_HAPPY
        printf("I'm just regular\n");
    #endif

    #ifdef EXTRA_HAPPY
        printf("I'm extra happy!\n");
    #else
        printf("I'm just regular\n");
    #endif


    #ifdef MODE_1
        printf("This is mode 1\n");
    #elifdef MODE_2
        printf("This is mode 2\n");
    #elifdef MODE_3
        printf("This is mode 3\n");
    #else
        printf("This is some other mode\n");
    #endif


    #include <stdio.h>
    #define EXTRA_HAPPY

    int main(void)
    {

    #ifdef EXTRA_HAPPY
    	printf("I'm extra happy!\n");
    #endif

    	printf("OK!\n");
    }

    ```

7.  **struct/union/enum 类型的前向声明** ：可以重复声明，主要用来消除头文件中的循环依赖：
    ```C
    // 前向声明，该类型实际在其它文件中定义。
    struct Rect;
    // 函数原型声明，使用前向声明的类型 struct Rect。
    bool is_in_rect(My_Point point, struct Rect rect);
    ```

前向声明的使用场景：

1.  自引用数据解构，如 stuct B 的成员应用 struct B 类型的指针；
2.  头文件中循环依赖，如 A.h 使用 B.h 中定义的 struct 类型；

访问前向声明的 struct/union/enum 的成员前，需要确保导入了它们的类型定义，否则编译失败：

```C
struct MyStruct;  // 前向声明

void myFunction() {
    struct MyStruct s;
    s.member = 10;  // 错误：不能访问前向声明的成员
}
```

不能在头文件重复定义的情况：一般在 c 源文件定义它们，在头文件中通过 extern 变量声明或
struct/enum/union 前向声明来使用它们：

1.  **变量和函数的定义** ：变量和函数的定义在同一个作用域内不能重复。例如：
    ```C
    int a;        // 定义
    int a = 10;   // 错误：重复定义

    void func(void) {
    	// 实现
    }

    void func(void) {
    	// 错误：重复定义
    }
    ```

2.  **结构体、联合体、枚举的重复定义** ：在同一个作用域内，不能重复定义同名但不同解构的结构体、联合体、枚举。例如：

    -   但是允许用 typedef 重复定义 struct/union/enum 类型，需要确保重复定义完全一致，否则编译报错。
    -   C 允许可以重复声明的 struct/union/enum 前向声明机制，在未定义它们时，在函数签名或类型定义中使用他们。

    <!--listend-->

    ```C
         struct MyStruct {
             int a;
         };

         struct MyStruct {
             int a;
         };  // 错误：重复定义
    ```

**Include Guard** ：防止头文件重复包含，使用 include guard 或 \`#pragma once\`。

-   使用 include gurad 可以消除同一个源文件多次 include 同一个头文件的问题，但是它不能消除 `不同源文件之间的重复包含同一个头文件` 。
-   Guard 宏名称惯例：对于用户头文件， `一般使用大写的头文件全称` ，对于系统库文件，一般是以 \_\_ 开头的宏名。
    ```C
    #ifndef MY_HEADER_H
    #define MY_HEADER_H
    // 内容
    #endif

    // 或者
    #pragma once
    // 内容
    ```

头文件中使用 #pragma once 来确保不会重复包含：

```C
// https://github.com/me-no-dev/OpenAI-ESP32/blob/master/src/OpenAI.h
#pragma once
#include "Arduino.h"
#include "cJSON.h"

class OpenAI_Completion;
class OpenAI_ChatCompletion;
class OpenAI_Edit;
class OpenAI_ImageGeneration;
```

宏定义比较特殊，它是 cpp 而非编译器处理的，而且是 `文件级别有效。头文件的 Once-Only 的 #ifndef 和
#ifdef 定义的宏 =只在定义它的源文件中生效` ，这是因为 C 预处理器在 `编译每个源文件之前独立地运行，处理在该文件中定义的所有预处理指令，包括宏定义` 。如果你在一个源文件中定义了一个宏，然后在另一个源文件中试图使用它，那么在编译那个尝试使用宏的源文件时，预处理器将不会知道那个宏的定义。所以，为了定义程序共享的宏定义（文件级别）， `应该在一个项目公共头文件中定义这个宏` ，然后在需要的源文件中包含这个头文件。

-   编译时可以使用 `-D 定义在所有源文件中都有效的宏名称` 。
-   可以在同一个源文件中定义相同的宏，最后的定义将被使用。


## <span class="section-num">2</span> 编译预处理指令 {#编译预处理指令}

\#include 支持相对路径：

```c
#include "mydir/myheader.h"
#include "../someheader.py"
```

\#include 和 #define 都是预处理指令，所以不以分号结尾：

```c
#define HELLO "Hello, world"
#define PI 3.14159
#define PI (3.14159) // 建议

// 在定义常量宏时可以不带 value，等效于 gcc -DEXTRA_HAPPY
#define EXTRA_HAPPY

#define SQR(x) (x) * (x)   // Better... but still not quite good enough!
#define SQR(x) ((x) * (x))   // Good!
```

条件编译：#if/#elif 后面支持条件表达式，必须使用常量（不能调用函数）。

```c
#ifndef MYHEADER_H  // First line of myheader.h
#define MYHEADER_H
int x = 12;
#endif  // Last line of myheader.h

#ifdef EXTRA_HAPPY
printf("I'm extra happy!\n");
#else
printf("I'm just regular\n");
#endif

#ifdef FOO
#if defined FOO
#if defined(FOO)   // Parentheses optional

#ifndef FOO
#if !defined FOO
#if !defined(FOO)   // Parentheses optional

#if __STDC_VERSION__ >= 1999901L

#include <stdio.h>

#define HAPPY_FACTOR 1

int main(void)
{

#if HAPPY_FACTOR == 0 // 条件表达式（常量）
	printf("I'm not happy!\n");
#elif HAPPY_FACTOR == 1
	printf("I'm just regular\n");
#else
	printf("I'm extra happy!\n");
#endif
	printf("OK!\n");
}
```

可以使用 #if 0 来注释一段代码块（代码块中可以包含注释）：

```c
#if 0
printf("All this code"); /* is effectively */
printf("commented out"); // by the #if 0
#endif
```

其他：

```c
#ifdef FOO
#if defined FOO
#if defined(FOO)   // Parentheses optional

#ifndef FOO
#if !defined FOO
#if !defined(FOO)   // Parentheses optional
```

取消宏定义 #undef：

```C
// #undef
#include <stdio.h>

int main(void)
{
#define GOATS

#ifdef GOATS
	printf("Goats detected!\n");  // prints
#endif

#undef GOATS  // Make GOATS no longer defined

#ifdef GOATS
	printf("Goats detected, again!\n"); // doesn't print
#endif
}
```

编译器内置的调试宏常量：
<span class="underline">+ <span class="underline">DATE</span></span>  The date of compilation—like when you’re compiling this file—in Mmm dd yyyy format
<span class="underline">+ <span class="underline">TIME</span></span>  The time of compilation in hh:mm:ss format
<span class="underline">+ <span class="underline">FILE</span></span>  A string containing this file’s name
<span class="underline">+ <span class="underline">LINE</span></span>  The line number of the file this macro appears on
<span class="underline">+ <span class="underline">func</span></span>  The name of the function this appears in, as a string128
<span class="underline">+ <span class="underline">STDC</span></span>  Defined with 1 if this is a standard C compiler
<span class="underline">+ <span class="underline">STDC_HOSTED</span></span>  This will be 1 if the compiler is a hosted implementation129, otherwise 0
<span class="underline">+ <span class="underline">STDC_VERSION</span></span>  This version of C, a constant long int in the form yyyymmL, e.g. 201710L

示例：

```C
#include <stdio.h>

int main(void)
{
	printf("This function: %s\n", __func__);
	printf("This file: %s\n", __FILE__);
	printf("This line: %d\n", __LINE__);
	printf("Compiled on: %s %s\n", __DATE__, __TIME__);
	printf("C Version: %ld\n", __STDC_VERSION__);
}
```

`__STDC_VERSION__`

-   C89/C90：没有定义 __STDC_VERSION\__。
-   C95：\__STDC_VERSION\_\_ 值为 199409L。
-   C99：\__STDC_VERSION\_\_ 值为 199901L。
-   C11：\__STDC_VERSION\_\_ 值为 201112L。
-   C17/C18：\__STDC_VERSION\_\_ 值为 201710L。

<!--listend-->

```c
#if __STDC_VERSION__ >= 1999901L
```

宏函数 #define：

```c
#define SQR(x) ((x) * (x))   // Good!
#define TRIANGLE_AREA(w, h) (0.5 * (w) * (h))
```

宏函数之间可以嵌套调用：

```c
#include <stdio.h>
#include <math.h>  // For sqrt()

#define QUADP(a, b, c) ((-(b) + sqrt((b) * (b) - 4 * (a) * (c))) / (2 * (a)))
#define QUADM(a, b, c) ((-(b) - sqrt((b) * (b) - 4 * (a) * (c))) / (2 * (a)))
#define QUAD(a, b, c) QUADP(a, b, c), QUADM(a, b, c)

int main(void)
{
	printf("2*x^2 + 10*x + 5 = 0\n");
	printf("x = %f or x = %f\n", QUAD(2, 10, 5));
}
```

可变参数：

```C
#include <stdio.h>

// Combine the first two arguments to a single number,
// then have a commalist of the rest of them:
#define X(a, b, ...) (10*(a) + 20*(b)), __VA_ARGS__

// 加前缀 # 表示替换为字符串
#define Y(...) #__VA_ARGS__

int main(void)
{
	printf("%d %f %s %d\n", X(5, 4, 3.14, "Hi!", 12));
	printf("%s\n", Y(1,2,3));  // Prints "1, 2, 3"
}
```

字符串化 #：

```c
#include <stdio.h>

#define STR(x) #x
#define PRINT_INT_VAL(x) printf("%s = %d\n", #x, x)

int main(void)
{
	int a = 5;
	PRINT_INT_VAL(a);  // prints "a = 5"

	printf("%s\n", STR(3.14159));
	// printf("%s\n", "3.14159");
}
```

参数连接 a ## b：

```c
#define CAT(a, b) a ## b // 中间需要有空格
printf("%f\n", CAT(3.14, 1592));   // 3.141592
```

多条语句的函数宏：

-   每条语句都以反斜杠结尾；
-   使用 do {} while(0) 包裹所有语句；

<!--listend-->

```c
#include <stdio.h>

#define PRINT_NUMS_TO_PRODUCT(a, b) do {		\
		int product = (a) * (b);		\
		for (int i = 0; i < product; i++) {	\
			printf("%d\n", i);		\
		}					\
	} while(0)

int main(void)
{
	PRINT_NUMS_TO_PRODUCT(2, 4);  // Outputs numbers from 0 to 7
}
```

ASSERT 宏：

```c
#include <stdio.h>
#include <stdlib.h>

#define ASSERT_ENABLED 1

#if ASSERT_ENABLED
#define ASSERT(c, m)							\
	do {								\
		if (!(c)) {						\
			fprintf(stderr, __FILE__ ":%d: assertion %s failed: %s\n", \
				__LINE__, #c, m);			\
			exit(1);					\
		}							\
	} while(0)
#else
#define ASSERT(c, m)  // Empty macro if not enabled
#endif

int main(void)
{
	int x = 30;
	ASSERT(x < 20, "x must be under 20");
}
```

\#error/#warning 指令：

```c
#ifndef __STDC_IEC_559__
#error I really need IEEE-754 floating point to compile. Sorry!
#endif
```

\#pragma 指令：

-   编译器遇到不认识的 #pragma 指令会直接忽略；
-   头文件中使用 #pragma once 来确保不会重复包含

<!--listend-->

```c
#pragma omp parallel for
for (int i = 0; i < 10; i++) { ... }

// 头文件中使用 once 来确保不会重复包含
#pragma once
```

重置 `__LINE__` 编号：后续从指定的编号开始计数；

```c
#line 300
// To override the line number and the filename:
#line 300 "newfilename"
```

Null 指令 #

```c
#ifdef FOO
#
#else
printf("Something");
#endif
```

\#embed Directive C23 新增：The gist of this is that you can include bytes of a file as integer
\#constants as if you’d typed them in.

```C
int a[] = {
#embed "foo.bin"
};
// 等效于：
int a[] = {11,22,33,44};
```


## <span class="section-num">3</span> 头文件搜索路径 {#头文件搜索路径}

\#include &lt;xx.h&gt; 不会搜索源文件所在的目录，而 #include "xx.h" 优先搜索源文件所在目录，然后是自定义目录和系统标准目录。

\#include "xx.h":

1.  搜索 include 该 header 文件的源文件所在目录下的 xx.h;
2.  -Idir1 -Idir2 指定的目录；
3.  -iquote dir1 -iquote dir2 指定的目录；
4.  系统标准目录；
5.  -idirafter dir1 -idirafter dir2 指定的目录；

\#include&lt;file&gt;

1.  -Idir1 -Idir2 指定的目录；
2.  -isystem dir1 -isystem dir2 指定的目录；
3.  系统标准目录；
4.  -idirafter dir1 -idirafter dir2 指定的目录；

注：-iquote 和 -isystem 的目录是各自专用的，不共用，而 -I 是共用的而且优先级高。

使用 `cpp -v` 命令查看搜索路径：

```shell
$ clang-cpp -v /dev/null -o /dev/null -I /tmp -iquote /usr -idirafter /etc -isystem /var
Homebrew clang version 16.0.4
Target: x86_64-apple-darwin22.4.0
Thread model: posix
InstalledDir: /usr/local/opt/llvm/bin
 (in-process)
 "/usr/local/Cellar/llvm/16.0.4/bin/clang-16" -cc1 -triple x86_64-apple-macosx13.0.0 -Wundef-prefix=TARGET_OS_ -Werror=undef-prefix -Wdeprecated-objc-isa-usage -Werror=deprecated-objc-isa-usage -E -disable-free -clear-ast-before-backend -disable-llvm-verifier -discard-value-names -main-file-name null -mrelocation-model pic -pic-level 2 -mframe-pointer=all -ffp-contract=on -fno-rounding-math -funwind-tables=2 -fcompatibility-qualified-id-block-type-checking -fvisibility-inlines-hidden-static-local-var -target-cpu penryn -tune-cpu generic -debugger-tuning=lldb -target-linker-version 857.1 -v -fcoverage-compilation-dir=/Users/zhangjun/docs -resource-dir /usr/local/Cellar/llvm/16.0.4/lib/clang/16 -iquote /usr -idirafter /etc -isystem /var -idirafter /home -I /tmp -isysroot /Library/Developer/CommandLineTools/SDKs/MacOSX13.sdk -internal-isystem /Library/Developer/CommandLineTools/SDKs/MacOSX13.sdk/usr/local/include -internal-isystem /usr/local/Cellar/llvm/16.0.4/lib/clang/16/include -internal-externc-isystem /Library/Developer/CommandLineTools/SDKs/MacOSX13.sdk/usr/include -fdebug-compilation-dir=/Users/zhangjun/docs -ferror-limit 19 -stack-protector 1 -fblocks -fencode-extended-block-signature -fregister-global-dtors-with-atexit -fgnuc-version=4.2.1 -fmax-type-align=16 -fcolor-diagnostics -D__GCC_HAVE_DWARF2_CFI_ASM=1 -o /dev/null -x c /dev/null
clang -cc1 version 16.0.4 based upon LLVM 16.0.4 default target x86_64-apple-darwin22.4.0
ignoring nonexistent directory "/Library/Developer/CommandLineTools/SDKs/MacOSX13.sdk/usr/local/include"
ignoring nonexistent directory "/Library/Developer/CommandLineTools/SDKs/MacOSX13.sdk/Library/Frameworks"
#include "..." search starts here:
 /usr
#include <...> search starts here:
 /tmp
 /var
 /usr/local/Cellar/llvm/16.0.4/lib/clang/16/include
 /Library/Developer/CommandLineTools/SDKs/MacOSX13.sdk/usr/include
 /Library/Developer/CommandLineTools/SDKs/MacOSX13.sdk/System/Library/Frameworks (framework directory)
 /etc
End of search list.
$
```

使用命令 `echo 'main(){}' | gcc -E -v -` 查看上面两个头文件搜索路径:

```shell
zhangjun@lima-ebpf-dev:/Users/zhangjun/docs$ echo 'main(){}' | gcc -E -v -
Using built-in specs.
COLLECT_GCC=gcc
OFFLOAD_TARGET_NAMES=nvptx-none:amdgcn-amdhsa
OFFLOAD_TARGET_DEFAULT=1
Target: x86_64-linux-gnu
Configured with: ../src/configure -v --with-pkgversion='Ubuntu 11.4.0-1ubuntu1~22.04' --with-bugurl=file:///usr/share/doc/gcc-11/README.Bugs --enable-languages=c,ada,c++,go,brig,d,fortran,objc,obj-c++,m2 --prefix=/usr --with-gcc-major-version-only --program-suffix=-11 --program-prefix=x86_64-linux-gnu- --enable-shared --enable-linker-build-id --libexecdir=/usr/lib --without-included-gettext --enable-threads=posix --libdir=/usr/lib --enable-nls --enable-bootstrap --enable-clocale=gnu --enable-libstdcxx-debug --enable-libstdcxx-time=yes --with-default-libstdcxx-abi=new --enable-gnu-unique-object --disable-vtable-verify --enable-plugin --enable-default-pie --with-system-zlib --enable-libphobos-checking=release --with-target-system-zlib=auto --enable-objc-gc=auto --enable-multiarch --disable-werror --enable-cet --with-arch-32=i686 --with-abi=m64 --with-multilib-list=m32,m64,mx32 --enable-multilib --with-tune=generic --enable-offload-targets=nvptx-none=/build/gcc-11-XeT9lY/gcc-11-11.4.0/debian/tmp-nvptx/usr,amdgcn-amdhsa=/build/gcc-11-XeT9lY/gcc-11-11.4.0/debian/tmp-gcn/usr --without-cuda-driver --enable-checking=release --build=x86_64-linux-gnu --host=x86_64-linux-gnu --target=x86_64-linux-gnu --with-build-config=bootstrap-lto-lean --enable-link-serialization=2
Thread model: posix
Supported LTO compression algorithms: zlib zstd
gcc version 11.4.0 (Ubuntu 11.4.0-1ubuntu1~22.04)
COLLECT_GCC_OPTIONS='-E' '-v' '-mtune=generic' '-march=x86-64'
 /usr/lib/gcc/x86_64-linux-gnu/11/cc1 -E -quiet -v -imultiarch x86_64-linux-gnu - -mtune=generic -march=x86-64 -fasynchronous-unwind-tables -fstack-protector-strong -Wformat -Wformat-security -fstack-clash-protection -fcf-protection -dumpbase -
ignoring nonexistent directory "/usr/local/include/x86_64-linux-gnu"
ignoring nonexistent directory "/usr/lib/gcc/x86_64-linux-gnu/11/include-fixed"
ignoring nonexistent directory "/usr/lib/gcc/x86_64-linux-gnu/11/../../../../x86_64-linux-gnu/include"
#include "..." search starts here:
#include <...> search starts here:
 /usr/lib/gcc/x86_64-linux-gnu/11/include
 /usr/local/include
 /usr/include/x86_64-linux-gnu
 /usr/include
End of search list.
# 0 "<stdin>"
# 0 "<built-in>"
# 0 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 0 "<command-line>" 2
# 1 "<stdin>"
main(){}
COMPILER_PATH=/usr/lib/gcc/x86_64-linux-gnu/11/:/usr/lib/gcc/x86_64-linux-gnu/11/:/usr/lib/gcc/x86_64-linux-gnu/:/usr/lib/gcc/x86_64-linux-gnu/11/:/usr/lib/gcc/x86_64-linux-gnu/
LIBRARY_PATH=/usr/lib/gcc/x86_64-linux-gnu/11/:/usr/lib/gcc/x86_64-linux-gnu/11/../../../x86_64-linux-gnu/:/usr/lib/gcc/x86_64-linux-gnu/11/../../../../lib/:/lib/x86_64-linux-gnu/:/lib/../lib/:/usr/lib/x86_64-linux-gnu/:/usr/lib/../lib/:/usr/lib/gcc/x86_64-linux-gnu/11/../../../:/lib/:/usr/lib/
COLLECT_GCC_OPTIONS='-E' '-v' '-mtune=generic' '-march=x86-64'
zhangjun@lima-ebpf-dev:/Users/zhangjun/docs$
```


### <span class="section-num">3.1</span> clang 头文件搜索路径 {#clang-头文件搜索路径}

-   MacOS 默认的系统头文件路径是 `/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/`, 使用命令可以查看 `xcrun --show-sdk-path`;
-   -isystem 可以[向 clang 搜索的 SYSTEM include search path 添加一个目录](https://clangd.llvm.org/guides/system-headers);
-   -isystem-after 和 -idirafter 是向 SYSTEM include search path 的末尾添加一个目录，两个选项都可以指定多次;
    -   `经过测试 -isystem-after 不生效` ，但是 [-idirafter 是可行的](https://gcc.gnu.org/onlinedocs/gcc/Directory-Options.html)。

clang 按照 -isysem/-idirafter `指定的顺序自左向右来依次搜索系统头文件目录` ，以找到的第一个为准。使用 `clang -v
-xc++ /dev/null -fsyntax-only` 命令可以查看 SYSTEM include search path：

```shell
$  xcrun --show-sdk-path
/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk

$ clang -v -xc++ /dev/null -fsyntax-only  -isystem /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/  -idirafter /Users/zhangjun/codes/ubuntu-5.15.0-75-headers
Homebrew clang version 16.0.4
Target: x86_64-apple-darwin22.4.0
Thread model: posix
InstalledDir: /usr/local/opt/llvm/bin
 (in-process)
 "/usr/local/Cellar/llvm/16.0.4/bin/clang-16" -cc1 -triple x86_64-apple-macosx13.0.0 -Wundef-prefix=TARGET_OS_ -Werror=undef-prefix -Wdeprecated-objc-isa-usage -Werror=deprecated-objc-isa-usage -fsyntax-only -disable-free -clear-ast-before-backend -disable-llvm-verifier -discard-value-names -main-file-name null -mrelocation-model pic -pic-level 2 -mframe-pointer=all -ffp-contract=on -fno-rounding-math -funwind-tables=2 -fcompatibility-qualified-id-block-type-checking -fvisibility-inlines-hidden-static-local-var -target-cpu penryn -tune-cpu generic -mllvm -treat-scalable-fixed-error-as-warning -debugger-tuning=lldb -target-linker-version 857.1 -v -fcoverage-compilation-dir=/Users/zhangjun/codes -resource-dir /usr/local/Cellar/llvm/16.0.4/lib/clang/16 -isystem /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/ -idirafter /Users/zhangjun/codes/ubuntu-5.15.0-75-headers -isysroot /Library/Developer/CommandLineTools/SDKs/MacOSX13.sdk -internal-isystem /usr/local/opt/llvm/bin/../include/c++/v1 -internal-isystem /Library/Developer/CommandLineTools/SDKs/MacOSX13.sdk/usr/local/include -internal-isystem /usr/local/Cellar/llvm/16.0.4/lib/clang/16/include -internal-externc-isystem /Library/Developer/CommandLineTools/SDKs/MacOSX13.sdk/usr/include -fdeprecated-macro -fdebug-compilation-dir=/Users/zhangjun/codes -ferror-limit 19 -stack-protector 1 -fblocks -fencode-extended-block-signature -fregister-global-dtors-with-atexit -fgnuc-version=4.2.1 -fcxx-exceptions -fexceptions -fmax-type-align=16 -fcolor-diagnostics -D__GCC_HAVE_DWARF2_CFI_ASM=1 -x c++ /dev/null
clang -cc1 version 16.0.4 based upon LLVM 16.0.4 default target x86_64-apple-darwin22.4.0
ignoring nonexistent directory "/Library/Developer/CommandLineTools/SDKs/MacOSX13.sdk/usr/local/include"
ignoring nonexistent directory "/Library/Developer/CommandLineTools/SDKs/MacOSX13.sdk/Library/Frameworks"
ignoring duplicate directory "/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include"
#include "..." search starts here:
#include <...> search starts here:
 /Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include
 /usr/local/opt/llvm/bin/../include/c++/v1
 /usr/local/Cellar/llvm/16.0.4/lib/clang/16/include
 /Library/Developer/CommandLineTools/SDKs/MacOSX13.sdk/System/Library/Frameworks (framework directory)
 /Users/zhangjun/codes/ubuntu-5.15.0-75-headers
End of search list.
$
```

可以使用环境变量 SDKROOT 修改 MacOS 的系统头文件路径, 这样所有搜索路径都以它为前缀：

```shell
$ export SDKROOT=/Users/zhangjun/codes/ubuntu-5.15.0-75-headers
$ xcrun --show-sdk-path
/Users/zhangjun/codes/ubuntu-5.15.0-75-headers

$ clang -v -xc++ /dev/null -fsyntax-only
Homebrew clang version 16.0.4
Target: x86_64-apple-darwin22.4.0
Thread model: posix
InstalledDir: /usr/local/opt/llvm/bin
 (in-process)
 "/usr/local/Cellar/llvm/16.0.4/bin/clang-16" -cc1 -triple x86_64-apple-macosx13.0.0 -Wundef-prefix=TARGET_OS_ -Werror=undef-prefix -Wdeprecated-objc-isa-usage -Werror=deprecated-objc-isa-usage -fsyntax-only -disable-free -clear-ast-before-backend -disable-llvm-verifier -discard-value-names -main-file-name null -mrelocation-model pic -pic-level 2 -mframe-pointer=all -ffp-contract=on -fno-rounding-math -funwind-tables=2 -fcompatibility-qualified-id-block-type-checking -fvisibility-inlines-hidden-static-local-var -target-cpu penryn -tune-cpu generic -mllvm -treat-scalable-fixed-error-as-warning -debugger-tuning=lldb -target-linker-version 857.1 -v -fcoverage-compilation-dir=/Users/zhangjun/docs -resource-dir /usr/local/Cellar/llvm/16.0.4/lib/clang/16 -isysroot /Users/zhangjun/codes/ubuntu-5.15.0-75-headers -internal-isystem /usr/local/opt/llvm/bin/../include/c++/v1 -internal-isystem /Users/zhangjun/codes/ubuntu-5.15.0-75-headers/usr/local/include -internal-isystem /usr/local/Cellar/llvm/16.0.4/lib/clang/16/include -internal-externc-isystem /Users/zhangjun/codes/ubuntu-5.15.0-75-headers/usr/include -fdeprecated-macro -fdebug-compilation-dir=/Users/zhangjun/docs -ferror-limit 19 -stack-protector 1 -fblocks -fencode-extended-block-signature -fregister-global-dtors-with-atexit -fgnuc-version=4.2.1 -fcxx-exceptions -fexceptions -fmax-type-align=16 -fcolor-diagnostics -D__GCC_HAVE_DWARF2_CFI_ASM=1 -x c++ /dev/null
clang -cc1 version 16.0.4 based upon LLVM 16.0.4 default target x86_64-apple-darwin22.4.0
ignoring nonexistent directory "/Users/zhangjun/codes/ubuntu-5.15.0-75-headers/usr/local/include"
ignoring nonexistent directory "/Users/zhangjun/codes/ubuntu-5.15.0-75-headers/usr/include"
ignoring nonexistent directory "/Users/zhangjun/codes/ubuntu-5.15.0-75-headers/System/Library/Frameworks"
ignoring nonexistent directory "/Users/zhangjun/codes/ubuntu-5.15.0-75-headers/Library/Frameworks"
#include "..." search starts here:
#include <...> search starts here:
 /usr/local/opt/llvm/bin/../include/c++/v1
 /usr/local/Cellar/llvm/16.0.4/lib/clang/16/include
End of search list.
$

$ gcc -v -xc++ /dev/null -fsyntax-only
Apple clang version 14.0.3 (clang-1403.0.22.14.1)
Target: x86_64-apple-darwin22.4.0
Thread model: posix
InstalledDir: /Library/Developer/CommandLineTools/usr/bin
ignoring nonexistent directory "/Users/zhangjun/codes/ubuntu-5.15.0-75-headers/usr/include/c++/v1"
 "/Library/Developer/CommandLineTools/usr/bin/clang" -cc1 -triple x86_64-apple-macosx13.0.0 -Wundef-prefix=TARGET_OS_ -Wdeprecated-objc-isa-usage -Werror=deprecated-objc-isa-usage -Werror=implicit-function-declaration -fsyntax-only -disable-free -clear-ast-before-backend -disable-llvm-verifier -discard-value-names -main-file-name null -mrelocation-model pic -pic-level 2 -mframe-pointer=all -fno-strict-return -ffp-contract=on -fno-rounding-math -funwind-tables=2 -fcompatibility-qualified-id-block-type-checking -fvisibility-inlines-hidden-static-local-var -target-cpu penryn -tune-cpu generic -mllvm -treat-scalable-fixed-error-as-warning -debugger-tuning=lldb -target-linker-version 857.1 -v -fcoverage-compilation-dir=/Users/zhangjun/docs -resource-dir /Library/Developer/CommandLineTools/usr/lib/clang/14.0.3 -isysroot /Users/zhangjun/codes/ubuntu-5.15.0-75-headers -stdlib=libc++ -internal-isystem /Library/Developer/CommandLineTools/usr/bin/../include/c++/v1 -internal-isystem /Users/zhangjun/codes/ubuntu-5.15.0-75-headers/usr/local/include -internal-isystem /Library/Developer/CommandLineTools/usr/lib/clang/14.0.3/include -internal-externc-isystem /Users/zhangjun/codes/ubuntu-5.15.0-75-headers/usr/include -internal-externc-isystem /Library/Developer/CommandLineTools/usr/include -Wno-reorder-init-list -Wno-implicit-int-float-conversion -Wno-c99-designator -Wno-final-dtor-non-final-class -Wno-extra-semi-stmt -Wno-misleading-indentation -Wno-quoted-include-in-framework-header -Wno-implicit-fallthrough -Wno-enum-enum-conversion -Wno-enum-float-conversion -Wno-elaborated-enum-base -Wno-reserved-identifier -Wno-gnu-folding-constant -fdeprecated-macro -fdebug-compilation-dir=/Users/zhangjun/docs -ferror-limit 19 -stack-protector 1 -fstack-check -mdarwin-stkchk-strong-link -fblocks -fencode-extended-block-signature -fregister-global-dtors-with-atexit -fgnuc-version=4.2.1 -fno-cxx-modules -no-opaque-pointers -fcxx-exceptions -fexceptions -fmax-type-align=16 -fcommon -fcolor-diagnostics -clang-vendor-feature=+disableNonDependentMemberExprInCurrentInstantiation -fno-odr-hash-protocols -clang-vendor-feature=+enableAggressiveVLAFolding -clang-vendor-feature=+revert09abecef7bbf -clang-vendor-feature=+thisNoAlignAttr -clang-vendor-feature=+thisNoNullAttr -mllvm -disable-aligned-alloc-awareness=1 -D__GCC_HAVE_DWARF2_CFI_ASM=1 -x c++ /dev/null
clang -cc1 version 14.0.3 (clang-1403.0.22.14.1) default target x86_64-apple-darwin22.4.0
ignoring nonexistent directory "/Users/zhangjun/codes/ubuntu-5.15.0-75-headers/usr/local/include"
ignoring nonexistent directory "/Users/zhangjun/codes/ubuntu-5.15.0-75-headers/usr/include"
ignoring nonexistent directory "/Users/zhangjun/codes/ubuntu-5.15.0-75-headers/System/Library/Frameworks"
ignoring nonexistent directory "/Users/zhangjun/codes/ubuntu-5.15.0-75-headers/Library/Frameworks"
#include "..." search starts here:
#include <...> search starts here:
 /Library/Developer/CommandLineTools/usr/bin/../include/c++/v1
 /Library/Developer/CommandLineTools/usr/lib/clang/14.0.3/include
 /Library/Developer/CommandLineTools/usr/include
End of search list.
$
```

1.  \#include &lt;xx&gt;: 只在系统标准路径搜索 xx, 不会搜索所在目录;
2.  \#include "dir/xx": 优先在包含该 #include 指令所在的目录搜索, 然后是自定义搜索路径, 然后是系统标准路径;

使用命令 `echo 'main(){}' | gcc -E -v -` 查看上面两个头文件搜索路径:

```shell
zhangjun@lima-ebpf-dev:/Users/zhangjun/docs$ echo 'main(){}' | gcc -E -v -
Using built-in specs.
COLLECT_GCC=gcc
OFFLOAD_TARGET_NAMES=nvptx-none:amdgcn-amdhsa
OFFLOAD_TARGET_DEFAULT=1
Target: x86_64-linux-gnu
Configured with: ../src/configure -v --with-pkgversion='Ubuntu 11.4.0-1ubuntu1~22.04' --with-bugurl=file:///usr/share/doc/gcc-11/README.Bugs --enable-languages=c,ada,c++,go,brig,d,fortran,objc,obj-c++,m2 --prefix=/usr --with-gcc-major-version-only --program-suffix=-11 --program-prefix=x86_64-linux-gnu- --enable-shared --enable-linker-build-id --libexecdir=/usr/lib --without-included-gettext --enable-threads=posix --libdir=/usr/lib --enable-nls --enable-bootstrap --enable-clocale=gnu --enable-libstdcxx-debug --enable-libstdcxx-time=yes --with-default-libstdcxx-abi=new --enable-gnu-unique-object --disable-vtable-verify --enable-plugin --enable-default-pie --with-system-zlib --enable-libphobos-checking=release --with-target-system-zlib=auto --enable-objc-gc=auto --enable-multiarch --disable-werror --enable-cet --with-arch-32=i686 --with-abi=m64 --with-multilib-list=m32,m64,mx32 --enable-multilib --with-tune=generic --enable-offload-targets=nvptx-none=/build/gcc-11-XeT9lY/gcc-11-11.4.0/debian/tmp-nvptx/usr,amdgcn-amdhsa=/build/gcc-11-XeT9lY/gcc-11-11.4.0/debian/tmp-gcn/usr --without-cuda-driver --enable-checking=release --build=x86_64-linux-gnu --host=x86_64-linux-gnu --target=x86_64-linux-gnu --with-build-config=bootstrap-lto-lean --enable-link-serialization=2
Thread model: posix
Supported LTO compression algorithms: zlib zstd
gcc version 11.4.0 (Ubuntu 11.4.0-1ubuntu1~22.04)
COLLECT_GCC_OPTIONS='-E' '-v' '-mtune=generic' '-march=x86-64'
 /usr/lib/gcc/x86_64-linux-gnu/11/cc1 -E -quiet -v -imultiarch x86_64-linux-gnu - -mtune=generic -march=x86-64 -fasynchronous-unwind-tables -fstack-protector-strong -Wformat -Wformat-security -fstack-clash-protection -fcf-protection -dumpbase -
ignoring nonexistent directory "/usr/local/include/x86_64-linux-gnu"
ignoring nonexistent directory "/usr/lib/gcc/x86_64-linux-gnu/11/include-fixed"
ignoring nonexistent directory "/usr/lib/gcc/x86_64-linux-gnu/11/../../../../x86_64-linux-gnu/include"
#include "..." search starts here:
#include <...> search starts here:
 /usr/lib/gcc/x86_64-linux-gnu/11/include
 /usr/local/include
 /usr/include/x86_64-linux-gnu
 /usr/include
End of search list.
# 0 "<stdin>"
# 0 "<built-in>"
# 0 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 0 "<command-line>" 2
# 1 "<stdin>"
main(){}
COMPILER_PATH=/usr/lib/gcc/x86_64-linux-gnu/11/:/usr/lib/gcc/x86_64-linux-gnu/11/:/usr/lib/gcc/x86_64-linux-gnu/:/usr/lib/gcc/x86_64-linux-gnu/11/:/usr/lib/gcc/x86_64-linux-gnu/
LIBRARY_PATH=/usr/lib/gcc/x86_64-linux-gnu/11/:/usr/lib/gcc/x86_64-linux-gnu/11/../../../x86_64-linux-gnu/:/usr/lib/gcc/x86_64-linux-gnu/11/../../../../lib/:/lib/x86_64-linux-gnu/:/lib/../lib/:/usr/lib/x86_64-linux-gnu/:/usr/lib/../lib/:/usr/lib/gcc/x86_64-linux-gnu/11/../../../:/lib/:/usr/lib/
COLLECT_GCC_OPTIONS='-E' '-v' '-mtune=generic' '-march=x86-64'
zhangjun@lima-ebpf-dev:/Users/zhangjun/docs$
```


## <span class="section-num">4</span> 内核头文件 {#内核头文件}

内核不能调用用户空间 C 库和其它任何不属于内核的库, 主要原因是速度和大小限制. 但内核也实现了大量 libc
的函数, 位于内核的根目录的 lib 目录下, 例如 lib/string.c 中定义了字符串操作函数, 调用时需要 include
对应的头文件 &lt;linux/string.h&gt; .

内核的头文件不能包含内核之外的其它头文件:

1.  基本的内核头文件位于内核根目录 include 目录下, 如 `#include <linux/i2c.h>` 的位于
    include/linux/i2c.h.
2.  平台架构相关的头文件位于 arch/&lt;arch&gt;/include/asm 和 arch/&lt;arch&gt;/include/uapi/asm 下, 包含这些头文件时, `以 asm/ 或 uapi/asm 为前缀`, 如 #include &lt;asm/ioctl.h&gt;
    -   asm/: 内核接口的一部分, 只被内核内部使用;
    -   uapi/asm: 是用户接口的一部分, 暴露到用户空间下, `可以被 glibc 使用`;
3.  include/asm-generic/ 和 include/uapi/asm-generic/ 目录:
    -   为所有平台共用的头文件,使用方式: #include &lt;asm-generic/irq.h&gt;,
    -   原则上不直接包含 asm-generic 头文件, 而是通过上面 arch 相关的头文件来包含.
4.  arch/&lt;arch&gt;/include/generated/asm/ 或 arch/&lt;arch&gt;/include/generated/uapi/asm/:
    -   实际上根据 include/asm-generic 目录下的头文件生成的.
5.  usr/include/ 为 kernel 导出的头文件, 主要给用户空间程序使用, 如编译 glibc 需要用到这些文件;
    -   后续的 make headers_install 命令负责生成导出的头文件.

ia64 和 x86_64 是不同的架构:

-   ia64 是 intel 64 bit Itanium architecture; (intel 专属);
-   x86_64 是 x86 兼容和扩展的架构;

在内核 arch 目录下, x86_64 是融合到 x86 目录下的.

示例: asm/types.h 包含了架构无关的 asm-generic/int-ll64.h 和架构相关的 uapi/asm/types.h;

```c
// /Users/zhangjun/codes/kernel/linux-5.10.y/include/uapi/linux/bpf.h
#include <linux/types.h>
#include <linux/bpf_common.h>

// /Users/zhangjun/codes/kernel/linux-5.10.y/include/linux/types.h
#include <uapi/linux/types.h>

// /Users/zhangjun/codes/kernel/linux-5.10.y/include/uapi/linux/types.h
#include <asm/types.h>

// /Users/zhangjun/codes/kernel/linux-5.10.y/arch/ia64/include/asm/types.h
#include <asm-generic/int-ll64.h> // include/asm-generic/int-ll64.h
#include <uapi/asm/types.h> // arch/ia64/include/uapi/asm/types.h
```

使用方式: <https://kernelnewbies.org/KernelHeaders>

1.  内核内部模块:使用内核根目录的 include/ 或 arch/\*/include/ 下的头文件.
2.  内核外置模块:
    -   传统方式: 使用 `/usr/src/linux` 目录下的头文件;
    -   新的方式(建议): 使用 `/lib/modules/${kver}/build` 目录下的 Makefile 来构建;
3.  用户空间程序, 如 glibc:
    -   一般情况下, 由各发行版提供的内核头文件 package 来安装, 如 glibc-devel, glibc-kernheaders or
        linux-libc-dev 等;
    -   使用 make headers_instal 来安装内核头文件, 然后重新构建 glibc 库;

内核的头文件可以使用 make headers_install 来安装到系统的 `/usr/include/` 目录中, 供系统 C 库如 glibc
来调用:

-   <https://www.linuxfromscratch.org/lfs/view/12.0/chapter05/linux-headers.html>

<!--listend-->

```shell
# 清理旧文件和依赖关系
make mrproper

make INSTALL_HDR_PATH=dest headers_install
find dest/include \( -name .install -o -name ..install.cmd \) -delete
cp -rv dest/include/* /usr/include

# 示例: http://smilejay.cn/2013/03/update-linux-headers/
[root@jay-linux kvm.git]# make headers_install
CHK     include/generated/uapi/linux/version.h
WRAP    arch/x86/include/generated/asm/clkdev.h
SYSHDR  arch/x86/syscalls/../include/generated/uapi/asm/unistd_32.h
SYSHDR  arch/x86/syscalls/../include/generated/uapi/asm/unistd_64.h
SYSHDR  arch/x86/syscalls/../include/generated/uapi/asm/unistd_x32.h
SYSTBL  arch/x86/syscalls/../include/generated/asm/syscalls_32.h
SYSHDR  arch/x86/syscalls/../include/generated/asm/unistd_32_ia32.h
SYSHDR  arch/x86/syscalls/../include/generated/asm/unistd_64_x32.h
SYSTBL  arch/x86/syscalls/../include/generated/asm/syscalls_64.h
HOSTCC  arch/x86/tools/relocs
HOSTCC  scripts/unifdef

INSTALL include/asm-generic (35 files)
INSTALL include/drm (15 files)
INSTALL include/linux/byteorder (2 files)
INSTALL include/linux/caif (2 files)
INSTALL include/linux/can (5 files)
INSTALL include/linux/dvb (8 files)
INSTALL include/linux/hdlc (1 file)
INSTALL include/linux/hsi (1 file)
INSTALL include/linux/isdn (1 file)
<省略部分信息...>
INSTALL include/asm (64 files)

[root@jay-linux include]# pwd
/usr/include
[root@jay-linux include]# mv asm asm_orig
[root@jay-linux include]# mv asm-generic asm-generic_orig
[root@jay-linux include]# mv linux linux_orig

[root@jay-linux kvm.git]# pwd
/root/kvm_demo/kvm.git
[root@jay-linux kvm.git]# cp -r usr/include/asm /usr/include/
[root@jay-linux kvm.git]# cp -r usr/include/asm-generic/ /usr/include/
[root@jay-linux kvm.git]# cp -r usr/include/linux /usr/include/
```

安装的 Linux API 头文件和目录介绍:

-   /usr/include/asm/\*.h: Linux API ASM 头文件(架构相关)
-   /usr/include/asm-generic/\*.h: Linux API ASM 通用头文件
-   /usr/include/drm/\*.h: Linux API DRM 头文件
-   /usr/include/linux/\*.h: Linux API Linux 头文件
-   /usr/include/misc/\*.h: The Linux API Miscellaneous Headers
-   /usr/include/mtd/\*.h: Linux API MTD 头文件
-   /usr/include/rdma/\*.h: Linux API RDMA 头文件
-   /usr/include/scsi/\*.h: Linux API SCSI 头文件
-   /usr/include/sound/\*.h: Linux API 音频头文件
-   /usr/include/video/\*.h: Linux API 视频头文件
-   /usr/include/xen/\*.h: Linux API Xen 头文件

编译 glibc 时依赖上面安装的内核头文件:
<https://www.linuxfromscratch.org/lfs/view/12.0/chapter05/glibc.html>

```shell
../configure                             \
    --prefix=/usr                      \
    --host=$LFS_TGT                    \
    --build=$(../scripts/config.guess) \
    --enable-kernel=4.14               \
    --with-headers=$LFS/usr/include    \
    libc_cv_slibdir=/usr/lib
```

--enable-kernel=4.14
: This tells Glibc to compile the library with support for 4.14 and later
    Linux kernels. Workarounds for older kernels are not enabled.

--with-headers=$LFS/usr/include
: This tells Glibc to compile itself against the headers recently
    installed to the $LFS/usr/include directory, so that it `knows exactly what features the kernel has
      and can optimize itself accordingly` .


## <span class="section-num">5</span> 头文件包含顺序 {#头文件包含顺序}

<https://google.github.io/styleguide/cppguide.html>

google C++ 编程风格对头文件的包含顺序作出如下指示：为了加强可读性和避免隐含依赖，应使用下面的顺序：

1.  C 标准库
2.  C++标准库
3.  其它库的头文件
4.  你自己工程的头文件

不过这里最先包含的是 `首选的头文件` ，即例如 a.cpp 文件中应该优先包含 a.h。先首选的头文件是 `为了发现隐藏依赖` ，同时确保头文件和实现文件是匹配的。

头文件 A.h 应该 #include `它依赖的所有其它头文件，如 B.h C.h` ，这样 c 源文件 A.c 中只需要感知和
\#include "A.h" 即可，从而避免显式的包含头文件 B.h/C.h。

假如你有一个 cc 文件（linux 平台的 cpp 文件后缀为 cc）是
google-awesome-project/src/foo/internal/fooserver.cc，那么它所包含的头文件的顺序如下：

-   工程头文件放到标准库头文件之后，这是因为这些头文件一般依赖于标准库头文件中定义的函数或声明。

<!--listend-->

```c
#include "foo/public/fooserver.h"  // 首选的头文件放在第一位

#include <sys/types.h>
#include <unistd.h>

#include <hash_map>
#include <vector>

#include "base/basictypes.h"
#include "base/commandlineflags.h"
#include "foo/public/bar.h"
```

隐含依赖又叫作隐藏依赖，即一个头文件依赖其它头文件，例如：

```c
//A.h
struct BS bs;
...

//B.h
struct BS{
....
};

//在 A.c 中，这样会报错
#include A.h
#include B.h

//先包含 B.h 就可以
#include B.h
#include A.h
```

这样就叫 `隐藏依赖` 。如果先包含 A.h 就可以发现隐藏依赖，所以各种规范都要求 `自身的头文件放在第一个` ，就能发现隐藏依赖。

对隐藏依赖的解决办法是：在 A.h 中包含 B.h，而不是在 A.c 中再包含，这样通过 A.h 屏蔽了它的依赖 B.h，在开发 A.c 时就不需要显式包含 B.h。

在包含头文件时应该加上头文件所在工程的文件夹名，即假如你有这样一个工程 base，里面有一个 logging.h，那么外部包含这个头文件应该这样写： `#include "base/logging.h"` ，而不是 `#include "logging.h"` 。

我们看到《Google C++ 编程风格指南》倡导原则背后隐藏的目的是：

1.  为了减少隐藏依赖，源文件应该 `最先包含其对应的头文件` ；
2.  之所以要 `将头文件所在工程目录列出` ，作用同命名空间一样，为了解决头文件重名问题。
