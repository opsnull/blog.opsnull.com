---
title: "Makefile-个人参考手册"
author: ["张俊(zj@opsnull.com)"]
date: 2024-05-24T00:00:00+08:00
lastmod: 2024-05-24T17:34:18+08:00
tags: ["make", "makefile", "Tools"]
categories: ["tool", "language"]
draft: false
---

这是我个人的 Makefile 个人参考手册。

<!--more-->

<details>
<summary>变更历史</summary>
<div class="details">

2024-05-24 Fri  首次创建
</div>
</details>

大型 C 程序一般由多个源文件组成，这些源文件分别编译，然后链接生成可执行程序。

将大型程序拆分为单独的源文件，单独编译为 obj 文件的好处是后续只需要重新编译修改的文件，而其它文件不需要重新编译，从而减少编译时间。但这带来一个管理文件变更和它的依赖编译的问题，手动处理这些问题容易出错，使用 Makefile 则可以自动解决这个问题。


## <span class="section-num">1</span> 原理 {#原理}

Makefile 规则中的 target 和 prerequests 的关系是：

1.  如果 target 不存在，则先检查各 prerequisite 是否存在，如果某个 prerequisite 不存在则执行对应
    target 规则的命令（命令执行完，不管是否生成该 prerequisite ，都认为该 prerequisite 被更新）；
2.  任意一个 prerequisite 被更新了（如修改时间比 target 晚），target 也必须被更新；
3.  targets 和 prerequisites 都可以是 pattern 或空格分割的 `多个文件名` ：
    -   多 target：makefile 会自动拆分生成为单 target 的等效规则；
4.  prerequisites 和 command 可以省略；
    -   当 prerequisites 为空时，如果 targets 文件不存在，则每次都会执行 command 命令；
5.  如果 command 没有生成 targetA 文件，则依赖 targetA 的对象 targetB，每次都会重新执行 targetA 的
    command 和重新生成 targetB；

假设 Makefile 中有一条规则 target A，执行规则 target A 的步骤：

1.  首先检查规则 A 的每个条件 P：
    -   如果存在以 P 为目标的规则 B，则执行规则 B。在执行规则 A 的过程中要执行规则 B，这是个 `递归的执行过程` 。
    -   如果找不到以 P 为目标的规则，并且文件 P 已存在，表示 P 不需要更新。
    -   如果找不到以 P 为目标的规则，并且文件 P 不存在，则报错退出。

2.  在检查完规则 A 的所有条件后，检查它的目标 T，如果属于以下情况之一，表示 T 需要更新就执行它的命令列表，执行完命令之后无论是否生成文件 T， `都认为 T 被更新过` 。
    -   文件 T 不存在。
    -   文件 T 存在，但是某个条件 P 是一个文件，该文件的修改时间（modify timestamp）比 T 晚。
        -   makefile 使用文件的 modify timestamp 来比较文件新旧；
    -   `某个条件 P 被更新过（并不一定生成文件 P）` 。

示例：make blah

```makefile
# 1. blash 不存在，2. 或者 blash.o 修改时间比 blash 新。都会重新生成 blah。
blah: blah.o
	cc blah.o -o blah # Runs third

blah.o: blah.c
	cc -c blah.c -o blah.o # Runs second

blah.c:
	echo "int main() { return 0; }" > blah.c # Runs first
```

示例：make some_file：

-   每次都先执行 other_file 下的命令，然后是 some_file 下的命令。这是因为some_file 依赖的 other_file
    不存在，故每次都执行两个 targe 下的命令。
-   执行完 other_file 下的命令后，不管 other_file 是否生成，都认为该条件已经执行完毕。

<!--listend-->

```makefile
some_file: other_file
	echo "This will always run, and runs second"
	touch some_file

other_file:
	echo "This will always run, and runs first"
```


## <span class="section-num">2</span> target {#target}

`对于多目标（如空格分割、pattern 匹配）的规则，make 会拆成几条单目标的规则来处理` ：

```makefile
target1 target2: preq1 preq2
	command $< -o $@
# 等效于
target1：preq1 preq2
	comand preq1 -o target1

target2: preq1 preq2
	command preq1 -o target2
```

target 的 prerequisite `可以写在一行或者拆开到多个规则中` ，这样就可以 `按需分步为` target 指定多个
prerequisite。写规则的目的是让 make 建立依赖关系图，不管怎么写，只要把所有的依赖关系都描述清楚了就行。

-   如果一个目标拆开写多条规则， `其中只有一条规则允许有命令列表，其他规则应该没有命令列表` ，否则 make 会

报警告并且 `采用最后一条规则的命令列表` 。

```makefile
main.o: main.h stack.h maze.h
main.o: main.c
	gcc -c main.c

# 等效于
main.o: main.c main.h stack.h maze.h
	gcc -c main.c
```

Double-Colon Rules：使用 :: 可以为一个 target 定义 `多个合并的 rule` ，这个 target 必须都使用 ::, 而不能 : 和 :: 混用，否则 make 命令执行报错；

```makefile
all: blah
blah:
	echo "hello"

blah::
	echo "hello again"

make all8

zj@a:~/go/src/github.com/opsnull/learn-by-doing/makefile$ make all8
Makefile:48: *** target file `blah' has both : and :: entries.  Stop.
```

.PHONY target：如果希望不管 target 文件是否存在，每次指定 make target 时都执行对应的命令，则需要将
target 标记为 .PHONY 伪目标的依赖，典型的使用场景是 clean 和 all：

```makefile
EXEC = mybin

all: ${EXEC}
.PHONY: all

${EXEC}: main.c main.h
	gcc -o ${EXECE} main.c

clean:
	rm -rf ${EXEC} *.o
.PHONY: clean  #按需为 .PHONY 伪目标添加新的 target

// 等效于
.PHONY: all clean
```

vpath：搜索指定 pattern 的 prerequisites 文件的目录：The vpath Directive: Use vpath to specify where
`some set of prerequisites exist`. The format is

```text
vpath <pattern> <directories, space/colon separated>
```

&lt;pattern&gt; can have a %, which matches any zero or more characters. You can also do this globallyish
with the `variable VPATH` ：

```makefile
vpath %.h ../headers ../other-directory  # % 只会匹配文件名, 而不是完整路径.

# Note: vpath allows blah.h to be found even though blah.h is never in the current directory
some_binary: ../headers blah.h  # 不用指定 blah.h 的具体路径, make 会按照 vpath 来进行搜索.
	touch some_binary

../headers:  # make 会查看 ../headers 目录是否存在,如果不存在则执行如下命令.
	mkdir ../headers

# We call the target blah.h instead of ../headers/blah.h, because that's the prereq that some_binary
# is looking for Typically, blah.h would already exist and you wouldn't need this.
blah.h: # make 会在 vpath 中搜索 blah.h 文件
	touch ../headers/blah.h

clean:
	rm -rf ../headers
	rm -f some_binary
```


## <span class="section-num">3</span> 隐式规则 {#隐式规则}

如果一个目标在 Makefile 中的所有规则都没有命令列表，make 会尝试在内建的 `隐含规则（Implicit Rule）` 中查找适用的规则。make 在解析 Makefile 时会把其中的规则及变量定义与 make 内建的隐含规则及变量定义融合在一起，用 make -p 命令可以查看所有这些规则和变量定义. 如果在一个没有 Makefile 的目录下使用 make -p
命令，则只显示 make 内建的隐含规则及变量定义。

隐式规则推导（$^ 表示所有 preqs）：

Compiling a C program
: `n.o` is made automatically from n.c with a command of the form

    ```text
    $(CC) -c $(CPPFLAGS) $(CFLAGS) $^ -o $@
    ```

Compiling a C++ program
: n.o is made automatically from n.cc or n.cpp with a command of the form

    ```text
    $(CXX) -c $(CPPFLAGS) $(CXXFLAGS) $^ -o $@
    ```

Linking a single object file
: `n` is made automatically from n.o by running the command

    ```text
    $(CC) $(LDFLAGS) $^ $(LOADLIBES) $(LDLIBS) -o $@
    ```

隐式推导规则自动使用的各环境变量：

AR
: Archive-maintaining program; default ‘ar’.

AS
: Program for compiling assembly files; default ‘as’.

CC
: Program for compiling C programs; default cc

CXX
: Program for compiling C++ programs; default g++

CFLAGS
: Extra flags to give to the C compiler

CXXFLAGS
: Extra flags to give to the C++ compiler

CPPFLAGS
: Extra flags to give to the C preprocessor

LDFLAGS
: Extra flags to give to compilers when they are supposed to invoke the linker

LDLIBS
: Library flags or names given to compilers when they are supposed to invoke the linker,
    ‘ld’. LOADLIBES is a deprecated (but still supported) alternative to LDLIBS. Non-library linker
    flags, such as -L, should go in the LDFLAGS variable.

通过隐式推导规则，只需要在 makefile 中声明对应的 preqs 而不需要再写 commands：

-   `command.o : defs.h command.h` ：make 根据目标是 .o 文件自动推导出编译命令：

    ```text
    $(CC) -c $(CPPFLAGS) $(CFLAGS) $^ -o $@
    ```

    等价于：

    ```text
    gcc -c $(CPPFLAGS) $(CFLAGS) defs.h command.h command.c -o command.o
    ```

<!--listend-->

```makefile
CC = gcc # Flag for implicit rules
CFLAGS = -g # Flag for implicit rules. Turn on debug info

# 使用隐式推导规则自动生成对应 command
main.o : defs.h
kbd.o : defs.h command.h
command.o : defs.h command.h
display.o : defs.h buffer.h
insert.o : defs.h buffer.h
search.o : defs.h buffer.h
files.o : defs.h buffer.h command.h
utils.o : defs.h

# 由于 make 为 target blah 和 blash.o 隐式自动生成规则，所以下面两个规则可以不用写。
#blah: blah.o
#blah.o: blah.c

# 如果 blah 除了依赖 blah.o 外还依赖其他 preq，如 blah.h，则需要写规则
blah: blah.o blah.h

blah.c:
	echo "int main() { return 0; }" > blah.c
```


## <span class="section-num">4</span> Command {#command}

Makefile 的 `command 前必须使用 TAB 来缩进` , 每一行都是 `单独的子 shell` ，所以如果要先 cd 再执行命令，需要再同一行上编写，分号分隔, 或者行尾用转义字符连成一行：

```makefile
all:
	cd ..
# The cd above does not affect this line, because each command is effectively run in a new shell
	echo `pwd`

#  This cd command affects the next because they are on the same line
	cd ..; echo `pwd`

# Same as above
	cd ..; \
	echo `pwd`
```

make 在执行 TAB 缩进的 command 时，会先显示命令再显示命令输出，可以在命令前添加 @ 来不显示执行的命令。

make 执行 command 如果出错（该命令的退出状态非0）就立刻终止，不再执行后续命令，但如果命令前面加了 -
字符（Hyphen），即使这条命令出错，make也会继续执行后续命令:

```makefile
all:
	@echo "This make line will not be printed"
	echo "But this will"

one:
# This error will be printed but ignored, and make will continue to run
	-false
	touch one
```

make 模式使用 POSIX `/usr/bin/sh` 来执行 command，可以通过 SHELL 变量来指定要使用的 shell；

```makefile
SHELL=/bin/bash

cool:
	echo "Hello from bash"
```


## <span class="section-num">5</span> 变量 {#变量}

Makefile 变量定义：变量值默认为字符串，故可以不加引号（单引号&amp;双引号均可）：

name := value
: 表示立即将 name 设置为 value(常用）；
    -   simply expanded (use :=) - like normal imperative programming -- only those defined so far get
        expanded

name = value
: 延迟将 name 设置为 value，具体延迟到使用 name 的时候计算 value 的值；
    -   recursive (use =) - only looks for the variables when the command is used, not when it's
        defined.

name ?= value
: 如果 name 未定义（不管是否为空），则使用 value（和 = 类似也是延迟展开）。

<!--listend-->

```makefile
# Recursive variable. This will print "later" below
one = one ${later_variable}
# Simply expanded variable. This will not print "later" below
two := two ${later_variable}

later_variable = later

all:
	echo $(one)  # 对于 one，在执行该命令时才展开替换，所以可以引用在 one 后面已经定义的变量 later_variable
	echo $(two)

files := file1 file2
some_file: $(files)
	echo "Look at this variable: " $(files)
	touch some_file
```

示例： 执行 make 命令时输出 Ah Huh？

```makefile
all:
	@echo $(foo)
foo = Ah $(bar)
bar = Huh?
```

虽然在 Makefile 中 bar 的定义写在 `foo = Ah $(bar)` 之后，而foo的定义写在echo$(foo)之后，最终还是能把
$(foo)展开成Ah $(bar)，把Ah $(bar)再展开成Ah Huh?。 关键要理解两点：

1.  Makefile 并不是从前到后顺序执行的，make 命令在执行前，会 `先解析整个 Makefile` ，生成各 target 完整的依赖树（因为 target 可能会多次赋值变量，多次关联 prerequisite，有多个同时生成 target 的规则等）。
2.  通过 = 号定义一个变量 ，如果=号右边有需要展开的形式（例如$(bar)），并不会在定义这个变量时立即展开，而是直到这个变量取值时（即要展开这个变量本身时）才进一步展开，也叫做递归地展开。

export name1 name2: 将 name1 和 name2 导出到 shell 和子 makefile 环境变量中；

-   执行 make 命令时的环境变量，可以直接在 makefile 中使用。

<!--listend-->

```makefile
# Run this with "export shell_env_var='I am an environment variable'; make"
all:
# Print out the Shell variable
	echo $$shell_env_var

# Print out the Make variable
	echo $(shell_env_var)


# shell_env_var 变量可以在 makefile 或子 shell 中使用。
shell_env_var=Shell env var, created inside of Make
export shell_env_var
all:
	echo $(shell_env_var)
	echo $$shell_env_var	# shell 展开环境变量
```

引用 makefile 变量： `$name, $(name) 或 ${name}` , 建议使用 $(name) 的形式，因为这种格式与 makefile
function 的调用方式一致，例如: OUTPUT := $(abspath .output)

-   由于 makefile 来调用 shell 来执行 COMMAND，所以在执行命令前，make 会对变量进行替换，而不 care 变量 quota 规则。
-   如果不想 make 来替换变量而是让 shell 来替换，可以使用 `$$var` 语法；
-   make 还会原样的把 command 中的引号传递给 shell；

<!--listend-->

```makefile
x := dude

all:
	echo $(x)
	echo ${x}
 # Bad practice, but works
	echo $x

a := one two # a is set to the string "one two"
b := 'one two' # Not recommended. b is set to the string "'one two'"
all:
	printf '$a'  # $a 会被 make 变量替换，所以输出 one two
	printf $b


make_var = I am a make variable
all9:
# Same as running "sh_var='I am a shell variable'; echo $sh_var" in the shell
	sh_var='I am a shell variable1'; echo $$sh_var  # make 不会替换 $$sh_var 而是传给 shell 变量 $sh_var
	sh_var2='I am a shell variable2'; echo '$$sh_var2' # make 不会替换 $$sh_var2, 也不会删除单引号，所以 shell 收到 '$sh_var2' ，shell 并不会变量替换；

	sh_var3='I am a shell variable3'; echo "$$sh_var3"  # 等效为 echo "$sh_var3", shell 会变量替换；

# Same as running "echo I am a make variable" in the shell
	echo $(make_var)
```

执行结果：

```shell
zj@a:~/go/src/github.com/opsnull/learn-by-doing/makefile$ make all9
sh_var='I am a shell variable1'; echo $sh_var  # make 不会替换 $sh_var 而是传给 shell 变量 h_var
I am a shell variable1
sh_var2='I am a shell variable2'; echo '$sh_var2' #
$sh_var2
sh_var3='I am a shell variable3'; echo "$sh_var3"
I am a shell variable3
echo I am a make variable
I am a make variable
```

在一个变量的定义中，从 `= 号或 :` 号右边的第一个非空白字符开始，直到注释或换行之前的所有字符都属于这个变量的值。

-   定义变量时，换行前的空格会保留，但是等号右边的空格会被删除，如果要生成一个空格，可以使用
    $(nullstring):

<!--listend-->

```makefile
with_spaces = hello   # with_spaces has many spaces after "hello"
after = $(with_spaces)there

nullstring =
space = $(nullstring) # Make a variable with a single space.

all:
	echo "$(after)"
	echo start"$(space)"end
```

在 `space := $(nullstring) # end of the line` 这个定义中，$(nullstring)展开为空，#号后边是注释，所以
space的值是$(nullstring)和#之间的那个空格。写注释是为了增加可读性，如果不写注释就换行，很难看出
$(nullstring)和换行之间有一个空格。

未定义的变量会被替换为空字符串：

```makefile
all:
# Undefined variables are just empty strings!
	echo $(nowhere)
```

使用 += 来给变量添加新的值，值之间用空格分割：

-   foov 是用 := 定义的，+= 号保持 := 号的特性；
-   如果变量还没有定义过就直接用 += 号赋值，那么 += 相当于 = 号；

<!--listend-->

```makefile
foov := start
foov += more

allv:
	echo $(foov)

zj@a:~/go/src/github.com/opsnull/learn-by-doing/makefile$ make allv
echo start more
start more
```

make 命令行参数重载：make option_one=hi

```makefile
# Overrides command line arguments
override option_one = did_override

# Does not override command line arguments
option_two = not_override
all:
	echo $(option_one)
	echo $(option_two)
```

除了 Makefile 全局变量外，还可以为特定 target 设置变量，这样该 target 的 command 应用该变量时，使用对应的值：

-   Target-specific variables： Variables can be set for specific targets

<!--listend-->

```makefile
all: one = cool

all:
	echo one is defined: $(one)

other:
	echo one is nothing: $(one)
```

通过 target-pattern，可以为一类 target 定义变量：

-   Pattern-specific variables： You can set variables for specific target patterns

<!--listend-->

```makefile
%.c: one = cool

blah.c:
	echo one is defined: $(one)

other:
	echo one is nothing: $(one)
```

自动变量[（Automatic Variables）](https://www.gnu.org/software/make/manual/html_node/Automatic-Variables.html) 指不需要手动赋值，在不同的上下文中自动取不同的值：

-   $@: 执行本条规则的单个 target 名称（常用）；
-   $?: 比 target 新的所有 prerequisite 的列表；
-   $^: 所有的 prerequisites，但消除重复的项目（常用）；
-   $&lt;: 当前第一个 prerequisites；（如果 prerequisite 是 pattern 生成的，则为第一个生成 prerequisite，一般在 %.o:%.c 的 command 中常用）

<!--listend-->

```makefile
all: hey hex

hex hey: one two
# Outputs "hey", since this is the target name
	@echo '$$@': $@

# Outputs all prerequisites newer than the target
	@echo '$$?': $?

# Outputs all prerequisites
	@echo '$$^': $^

	@echo '$$<': $<

	touch hey
one:
	@echo one
two:
	@echo two
```

执行：

```shell
zj@a:~/go/src/github.com/opsnull/learn-by-doing/makefile$ make all
one
two
$@: hey
$?:
$^: one two
$<: one
touch hey

$@: hex
$?: one two
$^: one two
$<: one
touch hey
```

如果要在参数中保留空格或使用逗号, 则需要使用变量机制:

```makefile
comma := ,
empty:=
space := $(empty) $(empty)
foo := a b c
bar := $(subst $(space),$(comma),$(foo))

all:
	@echo $(bar)
```

变量替换:

1.  $(patsubst pattern,replacement,text)
2.  $(text:pattern=replacement) # 快捷形式: 模式匹配;
3.  $(text:suffix=replacement) # 快捷形式: 后缀替换, 不包含 %
4.  $(text:=.sub)   # 给 text 中的每个对象添加 .sub 后缀;

<!--listend-->

```makefile
foosub := a.o b.o l.a c.o
onesub := $(patsubst %.o, %.c, $(foosub))

# This is a shorthand for the above
twosub := $(foosub:%.o=%.c)

# This is the suffix-only shorthand, and is also equivalent to the above.
threesub := $(foosub:.o=.c)

# 如果 foosub 变量存在, 则给每一个对象添加 .xx 后缀.
foursub := $(foosub:=.xx)
# 如果 nonsub 变量不存在, 则不添加 .xx 后缀, 返回空串.
nonsub := $(nonsub:=.xx)

allsub:
	echo $(onesub)
	echo $(twosub)
	echo $(threesub)
	echo $(foursub)
	echo "$(nonsub)x"

zj@a:~/go/src/github.com/opsnull/learn-by-doing/makefile$ make allsub
echo a.c b.c l.a c.c
a.c b.c l.a c.c
echo a.c b.c l.a c.c
a.c b.c l.a c.c
echo a.c b.c l.a c.c
a.c b.c l.a c.c
echo a.o.xx b.o.xx l.a.xx c.o.xx
a.o.xx b.o.xx l.a.xx c.o.xx
echo "x"
x
```


## <span class="section-num">6</span> 通配符 {#通配符}

targets/prerequests/变量值 value 都可以使用 wildcard：\* 和 %：

-   变量值 value 中的 \* 和 % 需要使用函数，否则不会被展开；
-   作为 targets 或 prerequests 时，匹配文件名，如果没有匹配则为原样值；

<!--listend-->

```makefile
# Print out file information about every .c file
print: $(wildcard *.c)
	ls -la  $?

thing_wrong := *.o # Don't do this! '*' will not get expanded
thing_right := $(wildcard *.o)

all: one two three four

# Fails, because $(thing_wrong) is the string "*.o"
one: $(thing_wrong)

# Stays as *.o if there are no files that match this pattern :(
two: *.o

# Works as you would expect! In this case, it does nothing.
three: $(thing_right)

# Same as rule three
four: $(wildcard *.o)
```

Static Pattern Rules：动态生成 target 和 prereq 规则：

-   用 target-pattern 中的 % 来匹配 targets 列表，对匹配的每一项，替换 prereq-patterns 中的 %；
-   对于 make 能生成的隐含规则, 不用添加具体的 command, 只是表明依赖关系

<!--listend-->

```makefile
# 语法
targets...: target-pattern: prereq-patterns ...
   commands


# 示例
objects = foo.o bar.o all.o
all: $(objects)

# These files compile via implicit rules
# Syntax - targets ...: target-pattern: prereq-patterns ...
# In the case of the first target, foo.o, the target-pattern matches foo.o and sets the "stem" to be "foo".
# It then replaces the '%' in prereq-patterns with that stem
$(objects): %.o: %.c

# make all.c： 当 all.c 不存在时，执行下面的 echo 命令，但执行完后，因为 all.c 已存在，故不再执行 touch 命令。
all.c:
	echo "int main() { return 0; }" > all.c

# 当 %.c 不存在时，执行下面的 touch 命令。
%.c:
	touch $@

clean:
	rm -f *.c *.o all
```

和 filter 连用：对 targes 中不同后缀文件使用不同的 command 生成规则：

-   $@ 表示当前生成的 target 名称；
-   $&lt; 表示匹配的第一个 prereq 名称；

<!--listend-->

```makefile
obj_files = foo.result bar.o lose.o
src_files = foo.raw bar.c lose.c

all: $(obj_files)
# Note: PHONY is important here. Without it, implicit rules will try to build the executable "all",
# since the prereqs are ".o" files.
.PHONY: all

# Ex 1: .o files depend on .c files. Though we don't actually make the .o file.
$(filter %.o,$(obj_files)): %.o: %.c
	echo "target: $@ prereq: $<"

# Ex 2: .result files depend on .raw files. Though we don't actually make the .result file.
$(filter %.result,$(obj_files)): %.result: %.raw
	echo "target: $@ prereq: $<"

%.c %.raw:
	touch $@

clean:
	rm -f $(src_files)
```

Pattern Rules，简化的静态模式规则（不需要指定 targets 列表）

```makefile
# Define a pattern rule that compiles every .c file into a .o file
%.o : %.c
	$(CC) -c $(CFLAGS) $(CPPFLAGS) $< -o $@

# Define a pattern rule that has no pattern in the prerequisites.  This just creates empty .c files
# when needed.
# 注：只有 .c 文件不存在时，才会执行 command
%.c:
	touch $@
```

递归执行 make 时，应该使用 $(MAKE) 而非 make，因为前者会带上 make flags：

```makefile
new_contents = "hello:\n\ttouch inside_file"
all:
	mkdir -p subdir
	printf $(new_contents) | sed -e 's/^ //' > subdir/makefile
	cd subdir && $(MAKE)

clean:
	rm -rf subdir
```

条件执行 ifeq/ifdef/else/endif：

```makefile
foo = ok

all:
# Conditional if/else
ifeq ($(foo), ok)
	echo "foo equals ok"
else
	echo "nope"
endif

# Check if a variable is empty
nullstring =
foo = $(nullstring) # end of line; there is a space here

all:
ifeq ($(strip $(foo)),)  # 逗号后的内容为空, 等效于: ifeq ($(strip $(foo)), '')
	echo "foo is empty after being stripped"
endif

ifeq ($(nullstring),)
	echo "nullstring doesn't even have spaces"
endif

#Check if a variable is defined
bar =
foo = $(bar)

all:
ifdef foo
	echo "foo is defined"
endif
ifndef bar
	echo "but bar is not"
endif
```


## <span class="section-num">7</span> 函数 {#函数}

函数: `$(fn, arguments)` or `${fn,arguments}` ，函数名称 fn, 多个 argument 之间, 均用逗号分割, 而且忽略逗号前后的空格:

```makefile
bar := ${subst not, totally     , "I am not superman"}
allbar:
	@echo "$(bar)x"

zj@a:~/go/src/github.com/opsnull/learn-by-doing/makefile$ make allbar
 I am totally supermanx
```

foreach 函数：

```makefile
foo := who are you
# For each "word" in foo, output that same word with an exclamation after
bar := $(foreach wrd,$(foo),$(wrd)!)

all:
# Output is "who! are! you!"
	@echo $(bar)
```

if 函数：

```makefile
foo := $(if this-is-not-empty,then!,else!)
empty :=
bar := $(if $(empty),then!,else!)

all:
	@echo $(foo)
	@echo $(bar)
```

call 函数：

```makefile
sweet_new_fn = Variable Name: $(0) First: $(1) Second: $(2) Empty Variable: $(3)

all:
# Outputs "Variable Name: sweet_new_fn First: go Second: tigers Empty Variable:"
	@echo $(call sweet_new_fn, go, tigers)
```

shell 函数：将结果的换行替换为空格：

```makefile
all:
	@echo $(shell ls -la) # Very ugly because the newlines are gone!
```

包含其它 makefile 的规则:

```makefile
# include 前的 - 表示忽略错误，如 filenames 都不存在的情况。
-include filenames ...
```

Multiline: The backslash ("\\") character gives us the ability to use multiple lines when the
commands are too long：用 \\ 转义的多行会被一个 sub shell 来执行, 否则每行的 command 都是一个子 shell;

```makefile
some_file:
	echo This line is too long, so \
		it is broken up into multiple lines
```


## <span class="section-num">8</span> make 命令 {#make-命令}

1.  -n：只打印要执行的命令，而不会真的执行命令（这称为 Dry Run），这个选项有助于我们检查Makefile写得是否正确，由于Makefile不是顺序执行的，用这个选项可以先看看命令的执行顺序，确认无误了再真正执行命令。
2.  -C： 切换到另一个目录执行那个目录下的 Makefile，
3.  在 make 命令行也可以用 `=或:` 定义变量，如果这次编译我想加调试选项 -g，但我不想每次编译都加 -g，可以在命令行定义 CFLAGS 变量而不必修改 Makefile：

    ```text
    make CFLAGS=-g
    ```

    1.  默认情况下 Makefile 中定义的变量会覆盖环境变量的定义，如果希望环境变量的定义覆盖 Makefile 中的定义可以用 -e 选项：make -e
    2.  在 make 的命令行选项中定义的变量优先级最高，会覆盖环境变量的定义和 Makefile 中的定义：

        ```text
        make foo=3
        ```


## <span class="section-num">9</span> GCC 动态生成 obj 的依赖 Makefile 规则 {#gcc-动态生成-obj-的依赖-makefile-规则}

对于复杂项目的 c 源文件，可能依赖多个自定义的头文件，手动写这些依赖比较复杂，这时可以借助 GCC 的-MMD
和 -MP 选项来在编译 obj 时自动生成该 obj 所依赖的头文件的 makefile 规则。

-MMD 和 -MP 是 GCC 编译器的两个选项，用于生成依赖文件，帮助管理源文件之间的依赖关系。以下是详细解释和实际例子：

MMD 选项：-MMD 选项用于生成依赖文件，但不包括系统头文件的依赖。这会创建一个 `以 .d 为扩展名的文件` ，包含该源文件的依赖信息。

-MP 选项：-MP 选项用于生成虚拟目标，以防止在头文件被删除后出现的 "No such file or directory" 错误。它为每个依赖的头文件生成一个伪目标，这些伪目标没有任何命令，从而确保即使头文件被删除，Makefile 也不会出错。实际例子

假设有以下项目结构：

```text
project/
├── src/
│   ├── main.c
│   ├── file1.c
│   ├── file1.h
│   ├── file2.c
│   └── file2.h
├── include/
│   └── project.h
└── Makefile
```

main.c

```C
#include "file1.h"
#include "file2.h"

int main() {
    func1();
    func2();
    return 0;
}
```

file1.c

```C
#include "file1.h"

void func1() {
    // Implementation
}
```

file1.h

```C
#ifndef FILE1_H
#define FILE1_H

void func1();

#endif
```

file2.c

```C
#include "file2.h"

void func2() {
    // Implementation
}
```

file2.h

```C
#ifndef FILE2_H
#define FILE2_H

void func2();

#endif
```

Makefile

```makefile
# Compiler and flags
CC = gcc
CFLAGS = -Wall -Iinclude
DEPFLAGS = -MMD -MP

# Directories
SRC_DIR = src
INCLUDE_DIR = include

# Source files
SRC_FILES = $(wildcard $(SRC_DIR)/*.c)

# Object files
OBJ_FILES = $(SRC_FILES:.c=.o) # 每一个 .c 对应一个 .o 文件

# Dependency files
DEP_FILES = $(OBJ_FILES:.o=.d)  # 每一个 .o 对应一个 .d 文件

# Output executable
OUTPUT = myprogram

# Default target
all: $(OUTPUT)

# Link object files
$(OUTPUT): $(OBJ_FILES)
	$(CC) -o $@ $^

# Compile source files with dependency generation
%.o: %.c
	$(CC) $(CFLAGS) $(DEPFLAGS) -c -o $@ $<  # 从 .c 生成 .o 时，自动生成对应的 .d 文件

# Include dependency files
-include $(DEP_FILES)  # 包含生成的 .d 文件中的规则

# Clean up
clean:
	rm -f $(SRC_DIR)/*.o $(SRC_DIR)/*.d $(OUTPUT)

.PHONY: all clean
```

生成的依赖文件内容示例

假设 main.c 包含以下内容：

```C
#include "file1.h"
#include "file2.h"
```

编译 main.c 时，-MMD -MP 选项会生成 main.d 文件，内容如下：

```makefile
src/main.o: src/main.c src/file1.h src/file2.h # 为某个 .o 生成对应的头文件依赖，这样就不需要手动维护了

src/file1.h:
src/file2.h:
```

这些依赖文件告诉 Make 在 main.c 或其包含的任何头文件发生变化时重新编译 main.o。虚拟目标 src/file1.h: 和 src/file2.h: 确保如果这些头文件被删除，不会导致 Make 出错。

解释

-   src/main.o: src/main.c src/file1.h src/file2.h 表示 main.o 依赖于 main.c、file1.h 和 file2.h。
-   src/file1.h: 和 src/file2.h: 是由 -MP 生成的虚拟目标，防止在头文件被删除后产生错误。

总结

通过使用 -MMD 和 -MP 选项，GCC 能够自动生成管理依赖关系的 .d 文件，确保 Makefile 在头文件发生变化或删除时能正确处理依赖关系。这大大简化了管理复杂项目的编译过程。


## <span class="section-num">10</span> 例子 {#例子}

下面是一个符合最佳实践的 Makefile 示例，演示了各种功能特性，包括编译、链接、依赖管理、清理和自动化生成依赖文件等。假设项目结构如下：

```text
project/
├── src/
│   ├── main.c
│   ├── file1.c
│   ├── file1.h
│   ├── file2.c
│   ├── file2.h
│   └── common.h
├── include/
│   └── project.h
├── lib/
│   ├── libfoo.a
│   └── libbar.a
├── ext_lib/
│   ├── libbaz.a
│   └── baz.h
└── Makefile
```

Makefile

```makefile
# Compiler and flags
CC = gcc
CFLAGS = -Wall -Iinclude -Isrc
DEPFLAGS = -MMD -MP

# Directories
SRC_DIR = src
INCLUDE_DIR = include
LIB_DIR = lib
EXT_LIB_DIR = ext_lib

# Source files
SRC_FILES = $(wildcard $(SRC_DIR)/*.c)

# Object files
OBJ_FILES = $(SRC_FILES:.c=.o)

# Dependency files
DEP_FILES = $(OBJ_FILES:.o=.d)

# External libraries
EXT_LIBS = -L$(EXT_LIB_DIR) -lbaz
LOCAL_LIBS = -L$(LIB_DIR) -lfoo -lbar

# Output executable
OUTPUT = myprogram

# Default target
all: $(OUTPUT)

# Link object files
$(OUTPUT): $(OBJ_FILES)
	$(CC) -o $@ $^ $(LOCAL_LIBS) $(EXT_LIBS)

# 显式指定链接的依赖顺序，这样 $^ 会按照指定的顺序包含所有 prerequisite。
# Link object files with specified order
$(OUTPUT): main.o a.o b.o
	$(CC) -o $@ $^ $(EXT_LIBS)

# Compile source files with dependency generation
%.o: %.c
	$(CC) $(CFLAGS) $(DEPFLAGS) -c -o $@ $<

# Include dependency files
-include $(DEP_FILES)

# Clean up
clean:
	rm -f $(SRC_DIR)/*.o $(SRC_DIR)/*.d $(OUTPUT)

# Additional PHONY targets
.PHONY: all clean
```

其他例子1:

```makefile
# https://makefiletutorial.com/#conditional-part-of-makefiles

# Thanks to Job Vranish (https://spin.atomicobject.com/2016/08/26/makefile-c-projects/)
TARGET_EXEC := final_program

BUILD_DIR := ./build
SRC_DIRS := ./src

# Find all the C and C++ files we want to compile
# Note the single quotes around the * expressions. The shell will incorrectly expand these otherwise, but we want to send the * directly to the find command.
SRCS := $(shell find $(SRC_DIRS) -name '*.cpp' -or -name '*.c' -or -name '*.s')

# Prepends BUILD_DIR and appends .o to every src file
# As an example, ./your_dir/hello.cpp turns into ./build/./your_dir/hello.cpp.o
OBJS := $(SRCS:%=$(BUILD_DIR)/%.o)

# String substitution (suffix version without %).
# As an example, ./build/hello.cpp.o turns into ./build/hello.cpp.d
DEPS := $(OBJS:.o=.d)

# Every folder in ./src will need to be passed to GCC so that it can find header files
INC_DIRS := $(shell find $(SRC_DIRS) -type d)
# Add a prefix to INC_DIRS. So moduleA would become -ImoduleA. GCC understands this -I flag
INC_FLAGS := $(addprefix -I,$(INC_DIRS))

# The -MMD and -MP flags together generate Makefiles for us!
# These files will have .d instead of .o as the output.
CPPFLAGS := $(INC_FLAGS) -MMD -MP

# The final build step.
$(BUILD_DIR)/$(TARGET_EXEC): $(OBJS)
	$(CXX) $(OBJS) -o $@ $(LDFLAGS)

# Build step for C source
$(BUILD_DIR)/%.c.o: %.c
	mkdir -p $(dir $@)
	$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@

# Build step for C++ source
$(BUILD_DIR)/%.cpp.o: %.cpp
	mkdir -p $(dir $@)
	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -c $< -o $@


.PHONY: clean
clean:
	rm -r $(BUILD_DIR)

# Include the .d makefiles. The - at the front suppresses the errors of missing
# Makefiles. Initially, all the .d files will be missing, and we don't want those errors to show up.
-include $(DEPS)
```

其他例子2:

```makefile
# https://github.com/iovisor/bcc/blob/master/libbpf-tools/Makefile

# SPDX-License-Identifier: (LGPL-2.1 OR BSD-2-Clause)
OUTPUT := $(abspath .output)
CLANG ?= clang
LLVM_STRIP ?= llvm-strip
BPFTOOL_SRC := $(abspath ./bpftool/src)
BPFTOOL_OUTPUT ?= $(abspath $(OUTPUT)/bpftool)
BPFTOOL ?= $(BPFTOOL_OUTPUT)/bootstrap/bpftool
LIBBPF_SRC := $(abspath ../src/cc/libbpf/src)
LIBBPF_OBJ := $(abspath $(OUTPUT)/libbpf.a)
LIBBLAZESYM_SRC := $(abspath blazesym/target/release/libblazesym.a)
INCLUDES := -I$(OUTPUT) -I../src/cc/libbpf/include/uapi
CFLAGS := -g -O2 -Wall
BPFCFLAGS := -g -O2 -Wall
INSTALL ?= install
prefix ?= /usr/local
ARCH ?= $(shell uname -m | sed 's/x86_64/x86/' | sed 's/aarch64/arm64/' \
			 | sed 's/ppc64le/powerpc/' | sed 's/mips.*/mips/' \
			 | sed 's/riscv64/riscv/' | sed 's/loongarch.*/loongarch/')
BTFHUB_ARCHIVE ?= $(abspath btfhub-archive)
ifeq ($(ARCH),x86)
CARGO ?= $(shell which cargo)
ifeq ($(strip $(CARGO)),) # 为空
USE_BLAZESYM ?= 0
else
USE_BLAZESYM ?= 1
endif
endif

ifeq ($(wildcard $(ARCH)/),)
$(error Architecture $(ARCH) is not supported yet. Please open an issue)
endif

BZ_APPS = \
	memleak \
	opensnoop \
	#

APPS = \
	bashreadline \
	bindsnoop \
	biolatency \
	biopattern \
	biosnoop \
	biostacks \
	biotop \
	bitesize \
	cachestat \
	capable \
	cpudist \
	cpufreq \
	drsnoop \
	execsnoop \
	exitsnoop \
	filelife \
	filetop \
	fsdist \
	fsslower \
	funclatency \
	gethostlatency \
	hardirqs \
	javagc \
	klockstat \
	ksnoop \
	llcstat \
	mdflush \
	mountsnoop \
	numamove \
	offcputime \
	oomkill \
	readahead \
	runqlat \
	runqlen \
	runqslower \
	sigsnoop \
	slabratetop \
	softirqs \
	solisten \
	statsnoop \
	syscount \
	tcptracer \
	tcpconnect \
	tcpconnlat \
	tcplife \
	tcppktlat \
	tcprtt \
	tcpstates \
	tcpsynbl \
	tcptop \
	vfsstat \
	wakeuptime \
	$(BZ_APPS) \
	#

# export variables that are used in Makefile.btfgen as well.
export OUTPUT BPFTOOL ARCH BTFHUB_ARCHIVE APPS

FSDIST_ALIASES = btrfsdist ext4dist nfsdist xfsdist
FSSLOWER_ALIASES = btrfsslower ext4slower nfsslower xfsslower
SIGSNOOP_ALIAS = killsnoop
APP_ALIASES = $(FSDIST_ALIASES) $(FSSLOWER_ALIASES) ${SIGSNOOP_ALIAS}

COMMON_OBJ = \
	$(OUTPUT)/trace_helpers.o \
	$(OUTPUT)/syscall_helpers.o \
	$(OUTPUT)/errno_helpers.o \
	$(OUTPUT)/map_helpers.o \
	$(OUTPUT)/uprobe_helpers.o \
	$(OUTPUT)/btf_helpers.o \
	$(OUTPUT)/compat.o \
	$(if $(ENABLE_MIN_CORE_BTFS),$(OUTPUT)/min_core_btf_tar.o) \
	#

ifeq ($(USE_BLAZESYM),1)
COMMON_OBJ += \
	$(OUTPUT)/libblazesym.a \
	$(OUTPUT)/blazesym.h \
	#
endif

define allow-override
  $(if $(or $(findstring environment,$(origin $(1))),\
            $(findstring command line,$(origin $(1)))),,\
    $(eval $(1) = $(2)))
endef

$(call allow-override,CC,$(CROSS_COMPILE)cc)
$(call allow-override,LD,$(CROSS_COMPILE)ld)

.PHONY: all
all: $(APPS) $(APP_ALIASES)

ifeq ($(V),1)
Q =
msg =
else
Q = @
msg = @printf '  %-8s %s%s\n' "$(1)" "$(notdir $(2))" "$(if $(3), $(3))";
MAKEFLAGS += --no-print-directory
endif

ifneq ($(EXTRA_CFLAGS),)
CFLAGS += $(EXTRA_CFLAGS)
endif
ifneq ($(EXTRA_LDFLAGS),)
LDFLAGS += $(EXTRA_LDFLAGS)
endif
ifeq ($(USE_BLAZESYM),1)
CFLAGS += -DUSE_BLAZESYM=1
endif

ifeq ($(USE_BLAZESYM),1)
LDFLAGS += $(OUTPUT)/libblazesym.a -lrt -lpthread -ldl
endif

.PHONY: clean
clean:
	$(call msg,CLEAN)
	$(Q)rm -rf $(OUTPUT) $(APPS) $(APP_ALIASES)

$(LIBBLAZESYM_SRC)::
	$(Q)cd blazesym && cargo build --release --features=cheader

$(OUTPUT)/libblazesym.a: $(LIBBLAZESYM_SRC) | $(OUTPUT)
	$(call msg,LIB,$@)
	$(Q)cp $(LIBBLAZESYM_SRC) $@

$(OUTPUT)/blazesym.h: $(LIBBLAZESYM_SRC) | $(OUTPUT)
	$(call msg,INC,$@)
	$(Q)cp blazesym/target/release/blazesym.h $@

$(OUTPUT) $(OUTPUT)/libbpf $(BPFTOOL_OUTPUT):
	$(call msg,MKDIR,$@)
	$(Q)mkdir -p $@

$(BPFTOOL): | $(BPFTOOL_OUTPUT)
	$(call msg,BPFTOOL,$@)
	$(Q)$(MAKE) ARCH= CROSS_COMPILE=  OUTPUT=$(BPFTOOL_OUTPUT)/ -C $(BPFTOOL_SRC) bootstrap

$(APPS): %: $(OUTPUT)/%.o $(COMMON_OBJ) $(LIBBPF_OBJ) | $(OUTPUT)
	$(call msg,BINARY,$@)
	$(Q)$(CC) $(CFLAGS) $^ $(LDFLAGS) -lelf -lz -o $@

ifeq ($(USE_BLAZESYM),1)
$(patsubst %,$(OUTPUT)/%.o,$(BZ_APPS)): $(OUTPUT)/blazesym.h
endif

$(patsubst %,$(OUTPUT)/%.o,$(APPS)): %.o: %.skel.h

$(OUTPUT)/%.o: %.c $(wildcard %.h) $(LIBBPF_OBJ) | $(OUTPUT)
	$(call msg,CC,$@)
	$(Q)$(CC) $(CFLAGS) $(INCLUDES) -c $(filter %.c,$^) -o $@

$(OUTPUT)/%.skel.h: $(OUTPUT)/%.bpf.o | $(OUTPUT) $(BPFTOOL)
	$(call msg,GEN-SKEL,$@)
	$(Q)$(BPFTOOL) gen skeleton $< > $@

$(OUTPUT)/%.bpf.o: %.bpf.c $(LIBBPF_OBJ) $(wildcard %.h) $(ARCH)/vmlinux.h | $(OUTPUT)
	$(call msg,BPF,$@)
	$(Q)$(CLANG) $(BPFCFLAGS) -target bpf -D__TARGET_ARCH_$(ARCH)	      \
		     -I$(ARCH)/ $(INCLUDES) -c $(filter %.c,$^) -o $@ &&      \
	$(LLVM_STRIP) -g $@

btfhub-archive: force
	$(call msg,GIT,$@)
	$(Q)[ -d "$(BTFHUB_ARCHIVE)" ] || git clone -q https://github.com/aquasecurity/btfhub-archive/ $(BTFHUB_ARCHIVE)
	$(Q)cd $(BTFHUB_ARCHIVE) && git pull

ifdef ENABLE_MIN_CORE_BTFS
$(OUTPUT)/min_core_btf_tar.o: $(patsubst %,$(OUTPUT)/%.bpf.o,$(APPS)) btfhub-archive | bpftool
	$(Q)$(MAKE) -f Makefile.btfgen
endif

# Build libbpf.a
$(LIBBPF_OBJ): $(wildcard $(LIBBPF_SRC)/*.[ch]) | $(OUTPUT)/libbpf
	$(call msg,LIB,$@)
	$(Q)$(MAKE) -C $(LIBBPF_SRC) BUILD_STATIC_ONLY=1		      \
		    OBJDIR=$(dir $@)libbpf DESTDIR=$(dir $@)		      \
		    INCLUDEDIR= LIBDIR= UAPIDIR=			      \
		    install

$(FSSLOWER_ALIASES): fsslower
	$(call msg,SYMLINK,$@)
	$(Q)ln -f -s $^ $@

$(FSDIST_ALIASES): fsdist
	$(call msg,SYMLINK,$@)
	$(Q)ln -f -s $^ $@

$(SIGSNOOP_ALIAS): sigsnoop
	$(call msg,SYMLINK,$@)
	$(Q)ln -f -s $^ $@

install: $(APPS) $(APP_ALIASES)
	$(call msg, INSTALL libbpf-tools)
	$(Q)$(INSTALL) -m 0755 -d $(DESTDIR)$(prefix)/bin
	$(Q)$(INSTALL) $(APPS) $(DESTDIR)$(prefix)/bin
	$(Q)cp -a $(APP_ALIASES) $(DESTDIR)$(prefix)/bin

.PHONY: force
force:

# delete failed targets
.DELETE_ON_ERROR:
# keep intermediate (.skel.h, .bpf.o, etc) targets
.SECONDARY:
```


## <span class="section-num">11</span> 参考 {#参考}

1.  <https://makefiletutorial.com/>
2.  [Learn Makefiles With the tastiest examples](https://makefiletutorial.com/)
3.  <https://seisman.github.io/how-to-write-makefile/overview.html>
