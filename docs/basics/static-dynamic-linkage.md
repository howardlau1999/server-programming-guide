# 静态链接与动态链接

在[上一节的最后](procedure.md#链接)，我们看到了 `ld` 命令中含有 `-static` 这个参数。这告诉链接器，要使用**静态链接**的方式生成最后的可执行文件。既然有**静态链接**，就会有**动态链接**。本节将带你了解两者的差别，并将带你了解如何创建和使用静态库和动态库。

## 静态链接

如果可执行文件以静态链接的方式链接，这就意味着在链接的时候，所有符号的二进制代码都会拷贝到最终的可执行文件中。例如，假如我们使用了 `printf` 这个函数，显然这个函数的二进制代码包含在系统库中，但是在链接的时候，也会拷贝到可执行文件里。

静态链接的做法比较简单，而且生成的二进制文件兼容性比较好，因为需要调用的函数都已经在文件里了，直接拷贝这个可执行文件到其他相同的操作系统执行，一般没有什么问题。当然，它也有缺点。

一是二进制的文件大小会变大。由于将所有用到的二进制代码都拷贝到了文件中，难免会让文件大小膨胀。

二是运行时占用内存会更多。我们知道，代码最终都是要加载到内存中执行的，二进制文件大小膨胀带来的一个后果就是运行的时候内存占用会更多。假如有十个运行的程序都用到了同一个库里的同样的函数，那么这十个程序会分别加载十次这个函数到内存中！

### 创建和使用静态链接库

在上一节中，我们见到了 `.a` 文件，这就是静态库文件，也是，下面让我们来试试看创建一个我们的静态库文件。假设我们的库提供了两个简单的函数，一个 `add` 函数和一个 `sub` 函数，用来做整数的加减法。我们创建两个文件，`add.cpp` 和 `sub.cpp`。

=== "add.cpp"
    
    ```cpp
    int add(int a, int b) {
        return a + b;
    }
    ```

=== "sub.cpp"

    ```cpp
    int sub(int a, int b) {
        return a - b;
    }
    ```

接下来，我们分别编译两个文件到目标文件。

```bash
$ g++ -c -o add.o add.cpp
$ g++ -c -o sub.o sub.cpp
$ ls
add.cpp  add.o  sub.cpp  sub.o
```

然后我们使用 `ar` 工具，将两个目标文件合并为一个 `.a` 文件。记得 `.a` 文件的命名遵循 `libxxx.a` 的格式，我们这里就起名为 `libmylib.a`。

```bash
$ ar r libmylib.a add.o sub.o
ar: creating libmylib.a
$ ls
add.cpp  add.o  libmylib.a  sub.cpp  sub.o
```

可以看到 `ar` 工具帮助我们创建了一个 `libmylib.a` 文件。我们用 `nm` 工具来看看里面包含了什么函数。

```bash
$ nm libmylib.a

add.o:
0000000000000000 T _Z3addii

sub.o:
0000000000000000 T _Z3subii
```

可以看到，一个 `.a` 文件其实就是几个 `.o` 文件的集合，并且我们的函数也包含在内了。

接下来，我们编写一个简单的程序来调用我们的库。在编写程序之前，我们需要提供一个头文件 `mylib.h` 来告诉使用我们的库的人，我们的库都提供了什么函数。头文件中不需要写出函数的定义，只需要声明函数的参数列表和返回值就可以了。

=== "mylib.h"

    ```cpp
    #ifndef __MYLIB_H__
    #define __MYLIB_H__
    int add(int a, int b);
    int sub(int a, int b);
    #endif
    ```

当其他人想要使用我们的库的时候，只需要在编写代码的时候，包含我们提供的头文件，就可以使用我们的函数了。

=== "main.cpp"

    ```cpp
    #include <iostream>
    #include "mylib.h"
    using namespace std;

    int main() {
        cout << "add(1, 2) = " << add(1, 2) << endl;
        cout << "sub(1, 2) = " << sub(1, 2) << endl;
        return 0;
    }
    ```

接下来，我们尝试编译我们的程序。请回忆上一节中关于预处理器和链接器的内容，我们需要分别使用 `-I` 选项告诉预处理器头文件在什么地方以及使用 `-L` 选项告诉链接器我们的库文件在什么地方。另外我们还需要告诉使用 `-l` 选项告诉链接器我们应该使用什么库。因此我们并不能简单地使用 `g++ main.cpp` 来编译程序，而是要提供正确的选项。另外，链接器对于链接选项的顺序也是有一定要求的，用到了库函数的程序需要放在库的前面。

```bash
$ g++ -I. main.cpp -L. -lmylib
$ ./a.out
add(1, 2) = 3
sub(1, 2) = -1
```

我们的程序编译通过，运行也正常，现在你已经学会如何创建并使用一个静态库了！

我们来看看使用静态库的二进制文件大小，方便等一下和动态链接作比较：

```bash
$ ls -l ./a.out
-rwxrwxr-x 1 howard howard 17496 Jul 11 17:39 ./a.out
```

可以看到此时的二进制大小是 17496 字节。

???+tip "使用 `#!cpp extern "C"` 导出函数"
    在[上一节的编译部分](procedure.md#编译)中，我们提到 C++ 编译器会对函数命名做混淆处理，在上面 `nm` 的输出中，我们也可以看到我们的 `add` 和 `sub` 函数名称前后附带了额外的字符。因为命名混淆的实现依赖于编译器实现，甚至不同版本的编译器混淆的方式也不一样。如果我们使用 C 语言程序来调用我们的库，或者使用了混淆方式不同的编译器，那么就会发生链接错误。
    
    为了解决这个问题，C++ 提供了 `#!cpp extern "C"` 的语法来禁用一个函数的命名混淆。也就是说，如果我们的 `sub.cpp` 这么写：

    ```cpp
    extern "C" int sub(int a, int b) {
        return a - b;
    }
    ```

    那么编译出来的目标文件将不会发生命名混淆。

    ```bash
    $ g++ -c -o sub.o sub.cpp
    $ nm sub.o
    0000000000000000 T sub
    ```
    
    需要注意的是，我们的头文件也需要对应地在函数声明上添加 `#!cpp extern "C"` 关键词，这样使用库的程序在编译时也不会将函数命名混淆，才能正确地链接。

    当然，在每一个函数前都加 `#!cpp extern "C"` 太麻烦，我们也可以用花括号的方式来声明花括号内的函数不要进行命名混淆。例如，我们可以在头文件中写：

    ```cpp
    #ifndef __MYLIB_H__
    #define __MYLIB_H__
    extern "C" {
    int add(int a, int b);
    int sub(int a, int b);
    }
    #endif
    ```

    这样所有的函数都不会进行命名混淆了。当然，这还有一个问题：C 语言并没有 `#!cpp extern "C"` 这个语法，使用我们的头文件会出错。这时候我们可以使用 `__cplusplus` 这个宏来判断我们的编译器是否 C++ 编译器，如果不是的话就不使用 `#!cpp extern "C"`。

    ```cpp
    #ifndef __MYLIB_H__
    #define __MYLIB_H__
    #ifdef __cplusplus
    extern "C" {
    #endif
    int add(int a, int b);
    int sub(int a, int b);
    #ifdef __cplusplus
    }
    #endif
    #endif
    ```

    在编写库的时候，最佳实践还是使用 `#!cpp extern "C"` 来保证我们的库有良好的兼容性。

    :warning: **注意：使用 `#!cpp extern "C"` 后，由于没有命名混淆，也就无法使用函数重载或模板等 C++ 功能。我们需要保证没有同名函数，否则会发生链接冲突。**

## 动态链接

为了解决静态链接带来的大小膨胀问题，人们提出了动态链接的方法，在编译的时候并不将库函数的二进制代码拷贝到可执行文件中，而是仅仅记录下来这个函数应该到什么文件中去寻找。运行的时候，需要调用这个函数的时候，再由操作系统来帮我们去加载对应的二进制代码。这种在运行时才真正加载二进制代码的方式就叫动态链接。而动态链接需要链接到共享库（Shared Library），Windows 上常见的 `.dll` 以及 Linux 上见到的 `.so` 文件就是程序运行的动态库了。这也是为什么有时候我们运行程序的时候，如果没有安装好运行时库，会提示“缺少 xxx.dll”而无法运行。

动态链接解决了静态链接的文件大小膨胀和内存占用增多的问题。一方面，可执行文件中不再包含二进制代码，另一方面，不同的进程可以共享同样的函数。例如，假如我们有十个进程都用到了 `printf` 这个函数，那么这十个进程可以共享内存中的同一份二进制代码，而不必重复加载代码到内存中，操作系统会处理共享的细节。

???+note "位置无关代码"

    由于动态链接，运行时二进制代码会被加载到内存中的哪个位置是完全不知道的。因此，动态链接库中的二进制代码必须被编译成位置无关的代码（Position-Independent Code）。例如，在静态链接中，我们可以指定一个变量 `a` 的地址为 `0x1234`，之后需要使用 `a` 的值的话只需要使用 `0x1234` 这个地址。但是动态链接的话，我们无法知道库会被加载到进程内存空间的哪个地址上，而且不同进程加载的地址可能完全不同。当然，我们可以让加载器在加载的时候把所有的地址都替换一遍，但如果每次加载都需要修改库里的地址的话，开销会非常大。这时候我们就需要库函数不能依赖绝对地址，而是需要将所有地址编码为相对的地址。例如，可以将地址编码为相对于目前指令的地址的偏移。这样的代码就能在不做修改的情况下被加载到内存中的任何地址了。

???+note "动态链接库运行时加载的过程"

    在动态链接的二进制里，存在着 PLT（Procedure Linkage Table）与 GOT（Global Offset Table）两张表。在程序启动时，操作系统会去将动态库映射到进程的地址空间中，并将函数的地址填入 GOT 表中。动态链接的函数在调用的时候，会查找 GOT 中符号对应的地址并跳转过去。
    
    为了加快程序启动的速度，操作系统并不会一次就将所有函数加载进来，而是在 GOT 表中，填入 PLT 表的地址，在第一次调用函数时，不会直接调用到函数本身，而是 PLT 的默认代码。PLT 的代码会去调用操作系统的函数，加载对应符号的地址，并改写 GOT 表，将对应的地址从 PLT 默认表项改为函数实际的地址，实现懒加载的过程。

    这也体现了一个经典原则：计算机的所有问题都可以通过增加一层间接性来解决。


当然，动态链接也有它自己的问题。首先，由于调用函数需要先查表再跳转，运行的性能会有一些损失。而且，由于动态库必须编译为 PIC，所有的绝对寻址指令都会变为相对寻址指令，可能会导致运行效率下降，不过现代 CPU 对 PIC 都有较好的支持，可以忽略不计。另外，由于是在程序运行时才加载的库，这就需要在发布程序时，也要将程序依赖的动态库一并发布，如果只发布二进制，就需要用户自己安装好对应的运行时库，例如，我们在 Steam 安装游戏后，往往还需要安装 `Microsoft VC++ Redistributable`，其实就是安装这个游戏需要的动态库。最后，由于各个程序依赖的动态库版本可能不同，在安装时如果互相覆盖了动态库，就会造成严重的问题，甚至可能导致其他程序无法正常运行。另外，在程序卸载的时候，如果删除了一个动态库，那也会造成其他程序无法运行。

???+note "DLL Hell"
    在 Windows 上，曾经有过著名的 [DLL Hell](https://en.wikipedia.org/wiki/DLL_Hell) 问题。在早期，计算机的内存还比较小，所以使用动态链接库十分受欢迎。同时，微软在早期的 Windows 版本中，允许并鼓励软件安装动态库时，直接安装到系统共享的目录中。许多安装器并不会检查已有的 DLL 版本。这就很容易发生动态库互相覆盖的问题。而且由于微软的设计失误，DLL 并没有很好的兼容性，一个小小的改动可能会使 DLL 文件发生巨大的变化，使得程序无法正常运行。假如程序 A 使用了 2.0 版本的 DLL，程序 B 使用了 1.0 版本的 DLL，如果先启动程序 B，那么就会加载旧版 DLL 到内存中，等程序 A 启动的时候，操作系统为了节省内存，会将 1.0 版本的 DLL 映射给程序 A，这样也会导致程序 A 无法正常运行。

    Windows 后来也解决了这个问题，一个方法是保护系统的 DLL 不被修改，另一个方法是使用 [WinSxS](https://en.wikipedia.org/wiki/Side-by-side_assembly) 技术，在系统里存放 DLL 的不同版本，尽管应用程序还是会用同一个名字来加载 DLL，操作系统会检查版本并加载正确的二进制代码。还有一个办法就是直接在 DLL 文件名加上版本号，这样不同版本的 DLL 就不再是同一个 DLL 了。

    Linux 解决 DLL Hell 的办法有两个，一个是 `.so` 文件会带有版本号。其次，大多数 Linux 软件发布的时候都会提供源代码，用户可以自己编译。最后，大多数 Linux 发行版都带有包管理软件来管理软件的依赖，在安装软件时就检查依赖问题，避免了 DLL Hell。

### 创建和使用动态链接库

我们还是以 `add.cpp` 和 `sub.cpp` 为例，创建动态库不需要我们对代码做任何修改，只需要修改编译命令即可。首先我们需要将两个文件编译为 PIC 目标文件。我们只需要在编译的时候添加 `-fPIC` 选项即可。为了方便你复制粘贴，我也将上面的代码拷贝一份到下面来。

=== "add.cpp"
    
    ```cpp
    int add(int a, int b) {
        return a + b;
    }
    ```

=== "sub.cpp"

    ```cpp
    int sub(int a, int b) {
        return a - b;
    }
    ```

=== "mylib.h"

    ```cpp
    #ifndef __MYLIB_H__
    #define __MYLIB_H__
    int add(int a, int b);
    int sub(int a, int b);
    #endif
    ```

=== "main.cpp"

    ```cpp
    #include <iostream>
    #include "mylib.h"
    using namespace std;

    int main() {
        cout << "add(1, 2) = " << add(1, 2) << endl;
        cout << "sub(1, 2) = " << sub(1, 2) << endl;
        return 0;
    }
    ```

```bash
$ g++ -fPIC -c -o add.o add.cpp
$ g++ -fPIC -c -o sub.o sub.cpp
```

和创建静态库不同，我们不使用 `ar` 工具来创建库文件，而是使用 `ld` 工具来创建动态库。动态库的命名规则和 `.a` 文件的命名规则一样，只是后缀名变为 `.so`。链接器链接时同样会尝试寻找名字相同的 `.so` 文件。例如，`-lmylib` 也会让链接器找到 `libmylib.so`。

```bash
$ ld -shared -o libmylib.so add.o sub.o
```

使用 `readelf` 工具可以看到，我们的 `libmylib.so` 是动态库类型了。并且 `nm` 工具也展示了其中包含的符号。可以看到，`.so` 文件并不是简单的 `.o` 文件的集合，其中的函数地址经过了修改。

```bash
$ readelf -h libmylib.so
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Shared object file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x1000
  Start of program headers:          64 (bytes into file)
  Start of section headers:          12872 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         9
  Size of section headers:           64 (bytes)
  Number of section headers:         13
  Section header string table index: 12
$ nm libmylib.so
0000000000003f40 d _DYNAMIC
0000000000001000 T _Z3addii
0000000000001018 T _Z3subii
```

和使用静态库一样，我们在程序中包含库的头文件，然后使用同样的命令编译即可。

```bash
$ g++ -I. main.cpp -L. -lmylib
$ ./a.out
./a.out: error while loading shared libraries: libmylib.so: cannot open shared object file: No such file or directory$ ./a.out
```

尽管编译成功了，但是运行的时候却提示找不到 `libmylib.so`，这是为什么呢？很快我们就会知道其中的原理，现在我们简单理解为系统默认并不会在我们的文件夹里寻找共享库，这个是由 `LD_LIBRARY_PATH` 环境变量决定的。我们将当前目录加入到这个环境变量中，然后再运行程序。

```bash
$ LD_LIBRARY_PATH=$(pwd):$LD_LIBRARY_PATH ./a.out
add(1, 2) = 3
sub(1, 2) = -1
```

程序正确地运行起来了！现在，你也学会了如何创建并使用一个动态库了。

我们来看看使用动态链接库的二进制文件大小：

```bash
$ ls -l ./a.out
-rwxrwxr-x 1 howard howard 17432 Jul 11 17:52 ./a.out
```

可以看到此时的二进制大小是 17432 字节，比使用静态链接库的二进制小了 64 字节。

???+note "使用 `ldd` 查看程序依赖哪些动态库"
    我们可以使用 `ldd` 命令来查看我们的程序依赖什么动态库。

    ```bash
    $ ldd a.out                            
        linux-vdso.so.1 (0x00007fff77fe7000)
        libmylib.so (0x00007fd011dc7000)
        libstdc++.so.6 => /lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007fd011bc5000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fd0119db000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007fd01188c000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fd011dd2000)
        libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007fd011871000)
    ```

???+tip
    如果你好奇尝试了的话，会发现即使上面编译目标文件的时候没有添加 `-fPIC` 选项，也能正常生成动态库。这是因为我们的函数并没有使用到全局的数据地址，事实上，那两个函数的参数会直接通过寄存器传递。为了演示 `-fPIC` 的必要性，我们可以修改 `sub.cpp` 为以下代码。

    === "sub.cpp"
        
        ```cpp
        int constant = 42;

        int sub(int a, int b) {
            return a - b;
        }

        int sub42(int a) {
            return a - constant;
        }
        ```

    此时我们不带 `-fPIC` 编译一遍 `sub.cpp`，并尝试使用 `ld` 生成动态库。

    ```bash
    $ g++ -c -o sub.o sub.cpp
    $ ld -shared -o libmylib.so sub.o add.o
    ld: sub.o: warning: relocation against `constant' in read-only section `.text'
    ld: sub.o: relocation R_X86_64_PC32 against symbol `constant' can not be used when making a shared object; recompile with -fPIC
    ld: final link failed: bad value
    ```

    可以看到此时 `ld` 提示 `constant` 这个变量无法重新定位，需要我们用 `-fPIC` 选项重新编译。
## 总结

本节介绍了静态链接与动态链接的区别以及如何创建和使用静态库和动态库。我们无法静态链接到一个动态库，也无法动态链接到一个静态库。如果使用静态库，那么只需要在**开发**的时候用到库文件。如果使用动态库，我们在**开发**和**运行**时都需要用到库文件。

在链接时，一部分库可能只提供了动态库，一部分库只提供了静态库，一部分则两者都有提供。链接时，链接器会自动选择合适的链接方式进行链接，也就是说一部分库可能以静态链接的方式链接到二进制中，一部分库则以动态链接方式链接到二进制中。