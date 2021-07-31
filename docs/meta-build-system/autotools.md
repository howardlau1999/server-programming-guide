# Autotools

Autotools 是 GNU 软件标准的元构建系统。

如果你试过从源码安装软件，那么一定对这几行命令非常熟悉：

```bash
./configure
make
make install
```

这个 `./configure` 脚本是怎么来的呢？答案就是它是用 Autotools 生成的。Autotools 发展历史悠久，可以从中学习到构建系统是如何一步步发展而来的。

## 从可移植性说起

尽管 C 语言号称是可移植的编程语言，但是 C 语言是高级语言中比较接近底层的，在使用 C 语言编程时，总会有一些函数会带来不可移植的问题，例如

- 有一些函数并不是每个平台都有提供，例如 `strtod`
- 有一些功能一样的函数，可能在不同平台的名称不一样，例如 `strchr` 或者 `index`
- 有一些功能和名称都一样的函数，可能函数原型不一样，例如 `#!cpp int setpgrp(void);` 和 `#!cpp int setpgrp(int, int);`
- 有一些函数可能原型一样，但是行为不一致，例如 `malloc(0)`
- 有一些函数完全一样，但是定义在了不同的头文件，例如 `string.h`、`strings.h` 和 `memory.h`
- 有一些函数可能声明完全一致，但是定义在了不同的库文件里，例如 `pow` 可能定义在 `libc.so` 或者 `libm.so` 里

没错，当有问题的时候，我们就可以引入一层间接性来解决它们：

- 用 `#!cpp #if` 和 `#!cpp #else` 来暴力判断平台然后编写不同的代码
- 用宏的方式来包装函数，并在编译的时候定义适合的宏
- 用函数的方式来包装函数，在编译的时候链接到不同的文件

其中第一种方式会使得源代码非常混乱，一般使用后两种方法。但是，无论使用哪一种方法，都一般会需要用户自己修改 Makefile 中的宏定义，或者手动编辑依赖等。这样安装软件会显得很麻烦，如果用户是新手的话可能就不想用了。所谓偷懒是科技进步的第一动力，程序员们自然会想：能不能写个程序帮用户自动生成对应平台的 Makefile？

因此，在大约 90 年代初期，人们开始为 GNU 包中的一些软件编写脚本，名字就叫 `configure`，来“猜测”用户平台的配置，然后自动生成一个 Makefile，这样用户只需要简单地运行几条命令，就能安装好软件了。

## 生成 configure 文件

当然，手写 `configure` 脚本对程序员来说也很麻烦，所以，没错，程序员们又写了一个生成 `configure` 脚本的工具，叫 `autoconf`，它本质上是一个宏处理器。不出意料，人们也发明了对应的 DSL。就好像 CMake 的脚本默认叫 `CMakeLists.txt`，`autoconf` 默认会去查找 `configure.ac` 这个文件（ac 就是 autoconf 的缩写），执行里面的命令，生成 `configure` 文件。

一个简单的 `configure.ac` 长这样：

=== "configure.ac"
    ```m4 linenums="1"
    AC_INIT([demo], [1.0], [bug-report@somewhere])
    AM_INIT_AUTOMAKE([foreign -Wall -Werror])
    AC_PROG_CC
    AC_CONFIG_HEADERS([config.h])
    AC_CONFIG_FILES([Makefile])
    AC_OUTPUT 
    ```

第一行类似于 `CMakeLists.txt` 的 `project`，声明了这个项目的名称以及版本。第二行则设置了一些编译参数。第三行表示需要检查 C 编译器。第四第五行表示需要生成的头文件以及 Makefile 文件名。第六行则表示正式输出所有的文件。

其中 `config.h` 文件会包含一些预定义的宏，例如 `PACKAGE_NAME` 等。可以看到 `configure.ac` 文件中并没有指定源代码，而是指定了 `Makefile`。不过你可能会好奇，`configure` 不就是为了生成 `Makefile` 吗，那这个 `Makefile` 又是从哪来的？第二行的 Automake 又是什么？

## 生成 Makefile 文件

没错，这个 `Makefile` 同样是从宏文件生成出来的，生成的工具叫 `automake`。这个工具会去查找 `Makefile.am`（am 是 automake 的缩写）文件，里面声明了程序的源文件以及生成的目标。下面是一个简单的程序和对应的 Makefile 模板：

=== "Makefile.am"

    ```m4
    bin_PROGRAMS = hello
    hello_SOURCES = main.c
    ```

=== "main.c"

    ```c
    #include <config.h>
    #include <stdio.h>

    int main() {
        puts("hello, world");
        puts("this is " PACKAGE_NAME ".");
        return 0;
    }
    ```

从变量名也能猜出来，`bin_PROGRAMS` 指定了 `bin` 文件夹下需要生成的目标，而 `hello_SOURCES` 则指定了生成 `hello` 这个目标所需要的源代码。就类似于 CMake 中的 `add_executable(hello main.c)`。

有了这两个文件，就足以构建这个简单的项目了。运行 `autoreconf`，就可以生成 Makefile 然后进行编译了。

```bash
$ ls
Makefile.am  configure.ac  main.c
$ autoreconf --install
configure.ac:3: installing './compile'
configure.ac:2: installing './install-sh'
configure.ac:2: installing './missing'
Makefile.am: installing './depcomp'
$ ./configure
checking for a BSD-compatible install... /usr/bin/install -c
checking whether build environment is sane... yes
checking for a thread-safe mkdir -p... /usr/bin/mkdir -p
# ...
configure: creating ./config.status
config.status: creating Makefile
config.status: creating config.h
config.status: executing depfiles commands
$ make              
make  all-am
make[1]: Entering directory '/home/howard/autoconfdemo'
gcc -DHAVE_CONFIG_H -I.     -g -O2 -MT main.o -MD -MP -MF .deps/main.Tpo -c -o main.o main.c
mv -f .deps/main.Tpo .deps/main.Po
gcc  -g -O2   -o hello main.o  
make[1]: Leaving directory '/home/howard/autoconfdemo'
$ ./hello    
hello, world
this is demo.
```

上面是一个简单的生成可执行文件的示例，如果想编译出静态库，则需要改动 `Makefile.am` 以及 `configure.ac`：

=== "Makefile.am"
    ```m4 hl_lines="1-2 6"
    lib_LIBRARIES = libhello.a
    libhello_a_SOURCES = hello.c hello.c

    bin_PROGRAMS = hello
    hello_SOURCES = main.c
    hello_LDADD = libhello.a
    ```

=== "configure.ac"
    ```m4 hl_lines="4-5"
    AC_INIT([demo], [1.0], [bug-report@somewhere])
    AM_INIT_AUTOMAKE([foreign -Wall -Werror])
    AC_PROG_CC
    AC_PROG_RANLIB
    AM_PROG_AR
    AC_CONFIG_HEADERS([config.h])
    AC_CONFIG_FILES([Makefile])
    AC_OUTPUT
    ```

=== "hello.c"
    ```c
    #include <stdio.h>
    #include <config.h>
    #include <hello.h>

    void sayhello() {
        puts("hello, world");
        puts("this is " PACKAGE_NAME ".");
    }
    ```

=== "hello.h"
    ```c
    #ifndef _HELLO_H_
    #define _HELLO_H_
    void sayhello();
    #endif /* ifndef _HELLO_H_ */
    ```

=== "main.c"
    ```c
    #include <config.h>
    #include <stdio.h>
    #include <hello.h>

    int main() {
        sayhello();
        return 0;
    }
    ```

同样执行 `configure` 重新生成 `Makefile` 文件，然后编译，就可以得到静态库和连接了静态库的程序了：

```bash
$ autoreconf --install && ./configure
# ...
$ make
make  all-am
make[1]: Entering directory '/home/howard/autoconfdemo'
gcc -DHAVE_CONFIG_H -I.     -g -O2 -MT main.o -MD -MP -MF .deps/main.Tpo -c -o main.o main.c
mv -f .deps/main.Tpo .deps/main.Po
gcc -DHAVE_CONFIG_H -I.     -g -O2 -MT hello.o -MD -MP -MF .deps/hello.Tpo -c -o hello.o hello.c
mv -f .deps/hello.Tpo .deps/hello.Po
rm -f libhello.a
ar cru libhello.a hello.o hello.o 
ar: `u' modifier ignored since `D' is the default (see `U')
ranlib libhello.a
gcc  -g -O2   -o hello main.o libhello.a 
make[1]: Leaving directory '/home/howard/autoconfdemo'
$ ./hello
hello, world
this is demo.
```

## 检查平台特性

在运行 `./configure` 过程中，我们会看到类似这样的输出：

```
checking for mkdir... yes
checking for abcdefg... no
```

这就是 `configure` 脚本在检查系统的环境特性。它的工作原理同样是使用宏，然后在执行过程中，动态生成一个源程序，然后尝试编译等操作，去检查某些类型或者函数是否被定义。当然我们也可以自定义宏去检查更复杂的平台相关特性。

例如，我们想知道系统里有没有 `mkdir` 这个函数，就可以使用 `AC_CHECK_FUNCS([mkdir])`。如果有这个函数的话，那么在 `config.h` 中，就会定义 `#!cpp #define HAVE_MKDIR 1`，我们就可以使用这个宏定义进行条件编译了。

## 分发源代码

那么，我们平时下载的 `xxx.tar.gz` 是怎么来的呢？我们可以手动将生成的 `configure` 等各种文件连同源代码一起打包。不过，Autotools 生成的 Makefile 里面已经内置了这样的脚本，我们只需要执行 `make dist`，就可以自动生成一个 `<package>-<version>.tar.gz` 的压缩包，只需要将这个压缩包分发给用户，用户解压之后就能用经典的三部曲编译使用我们的程序了。

```bash
$ make dist
# ...
$ ls *.tar.gz
demo-1.0.tar.gz
```
## autoreconf 背后的故事

上面用到了 `autoreconf`，这是一个帮助我们运行各种工具生成构建所需要的文件的包装工具。就跟 `g++` 是包装器一样，`autoreconf` 背后会帮我们调用 `autoconf`、`automake` 等一系列工具来生成构建文件。下面简单介绍一下实际生成文件的程序。

|程序名|作用|
|----|----|
|aclocal|扫描 `configure.ac` 中使用的第三方宏，并将它们的定义收集到 `aclocal.m4` 中|
|autoconf|根据 `configure.ac` 生成 `configure` 脚本|
|autoheader|根据 `configure.ac` 生成 `config.h.in` 文件|
|automake|根据 `configure.ac` 以及 `Makefile.am` 生成 `Makefile.in` 文件|
|autom4te|M4（宏处理器）的 autoconf 驱动程序，所有处理 `configure.ac` 的程序都是用这个程序来处理宏的|