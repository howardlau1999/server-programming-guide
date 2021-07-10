# 从源代码到二进制

首先来看一个简单的程序

=== "hello.cpp"

    ```cpp
    #include <iostream>
    using namespace std;

    int main() {
        cout << "Hello World!" << endl;
        return 0;
    }
    ```

我们都知道，我们只需要运行 `g++ hello.cpp` 就可以将这个 C++ 源代码文件编译为可执行文件 `a.out` 了。其实，`g++` 只是一个驱动程序（Driver），它背后发生了一系列复杂的操作，调用了许多不同的程序。

## 预处理

在编译开始之前，预处理器会处理源文件中的宏定义（例如 `#!cpp #define`、`#!cpp #include`、`#!cpp #ifdef` 等），对源代码进行展开或者替换。例如，对于 `#!cpp #include <iostream>`，预处理器会直接将 `iostream` 这个文件原封不动地拷贝到源文件中。然而，我们的源代码目录中并没有 `iostream` 这个文件，预处理器是怎么知道去哪里查找这个文件呢？

实际上，预处理器并不知道我们的头文件都在哪里，需要我们手动提供包含路径（Include Path）。对于 `g++` 而言，就是 `-I` 选项。预处理器会在我们提供的路径中依次搜索，直到找到头文件为止。当然，像 `iostream` 这种常用的系统头文件，如果每次都需要我们手动指定查找目录，将会非常繁琐，所以 `g++` 已经内置了许多常用的包含路径，同时，包含路径默认还会包括源文件所在的目录。

这一步的输出是经过宏展开之后的源代码文件，并不是所有语言都像 C/C++ 一样有类似文本替换的预处理步骤。

???+note
    在 C/C++ 中，预处理程序是 `cpp`，全称是 **C** **P**re**P**rocessor。可以运行 `cpp hello.cpp` 查看预处理器的输出。输出可能比较长，可以将输出重定向到一个文件方便查看。

???+info
    使用 `-E` 选项，可以使 `g++` 只输出预处理结果而不进行进一步的编译。可以运行 `g++ -E hello.cpp` 查看输出，和直接运行 `cpp` 的结果比较。
### 试试看

下面我们来试试看如果包含路径不在默认提供的范围里，我们应当如何让预处理器找到文件。首先，创建两个文件夹，分别是 `src` 和 `include`。在里面分别放入以下文件：

=== "src/a.cpp"

    ```cpp
    #include "a.h"
    
    int main() {
        return a;
    }
    ```

=== "include/a.h"

    ```cpp
    int a = 1;
    ```

首先我们直接运行 `g++ -E src/a.cpp`，可以看到，这时候预处理器表示自己找不到 `a.h` :thinking:：

```bash
$ g++ -E src/a.cpp
# 1 "src/a.cpp"
# 1 "<built-in>"
# 1 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 1 "<command-line>" 2
# 1 "src/a.cpp"
src/a.cpp:1:10: fatal error: a.h: No such file or directory
    1 | #include "a.h"
      |          ^~~~~
compilation terminated.
```

这时，就需要我们通过 `-I` 选项额外提供查找目录。这时候预处理器就知道去哪里找头文件了，并且 `a.h` 中的内容也被正确地复制进来了:laughing:。

```bash
$ g++ -E -Iinclude src/a.cpp
# 1 "src/a.cpp"
# 1 "<built-in>"
# 1 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 1 "<command-line>" 2
# 1 "src/a.cpp"
# 1 "include/a.h" 1
int a = 1;
# 2 "src/a.cpp" 2

int main() {
    return a;
}
```
!!!note
    如果需要指定多个查找目录，需要提供多个 `-I` 选项，预处理器会依次查找。例如 `g++ -Iinclude1 -Iinclude2`。
## 编译

源文件经过预处理后，便可以调用编译器对源文件进行编译了。简单来讲，每一个 `.cpp` 文件可以被称为编译单元（Compilation Unit）。每一个编译单元都可以独立进行编译。编译器的工作是解析源文件，经过词法分析、语法分析、中间代码生成、代码优化等一系列复杂的操作，最终生成一个目标平台的汇编文件。

这一步的输出是汇编文件，不同平台所使用的汇编语言不一样，这里以 `x86-64` 平台为例。

???+note
    `g++` 所使用的 C++ 编译器为 `cc1plus`，这个程序并不在默认的 `PATH` 中，而是在库文件中。如果你使用的是 `g++` 9，那么它就在 `/usr/lib/gcc/x86_64-linux-gnu/9/cc1plus`。

### 试试看

我们首先将之前的 `hello.cpp` 进行预处理，保存到 `hello.preprocessed.cpp` 中，然后调用 `cc1plus` 将 `hello.preprocessed.cpp` 编译到汇编文件 `hello.preprocessed.s`。

```bash
$ cpp hello.cpp hello.preprocessed.cpp
$ /usr/lib/gcc/x86_64-linux-gnu/9/cc1plus hello.preprocessed.cpp
# ... 省略很多输出
Analyzing compilation unit
Performing interprocedural optimizations
 <*free_lang_data> <visibility> <build_ssa_passes> <opt_local_passes> <remove_symbols> <targetclone> <free-fnsummary>Streaming LTO
 <whole-program> <hsa> <fnsummary> <inline> <free-fnsummary> <single-use> <comdats>Assembling functions:
 <materialize-all-clones> <simdclone> int main() void __static_initialization_and_destruction_0(int, int) void _GLOBAL__sub_I_main()
Time variable                                   usr           sys          wall               GGC
 phase setup                        :   0.00 (  0%)   0.00 (  0%)   0.01 (  2%)    1471 kB (  5%)
 phase parsing                      :   0.19 ( 86%)   0.24 ( 77%)   0.48 ( 81%)   25530 kB ( 83%)
 phase lang. deferred               :   0.02 (  9%)   0.06 ( 19%)   0.08 ( 14%)    3548 kB ( 12%)
 phase opt and generate             :   0.01 (  5%)   0.01 (  3%)   0.02 (  3%)     289 kB (  1%)
 |name lookup                       :   0.00 (  0%)   0.05 ( 16%)   0.05 (  8%)    1463 kB (  5%)
 |overload resolution               :   0.02 (  9%)   0.04 ( 13%)   0.02 (  3%)    1788 kB (  6%)
 callgraph optimization             :   0.00 (  0%)   0.00 (  0%)   0.01 (  2%)       0 kB (  0%)
 ipa inlining heuristics            :   0.00 (  0%)   0.01 (  3%)   0.00 (  0%)       0 kB (  0%)
 preprocessing                      :   0.01 (  5%)   0.02 (  6%)   0.04 (  7%)     377 kB (  1%)
 parser (global)                    :   0.04 ( 18%)   0.07 ( 23%)   0.18 ( 31%)    8434 kB ( 27%)
 parser struct body                 :   0.04 ( 18%)   0.02 (  6%)   0.05 (  8%)    5710 kB ( 19%)
 parser function body               :   0.03 ( 14%)   0.02 (  6%)   0.00 (  0%)    1043 kB (  3%)
 parser inl. func. body             :   0.02 (  9%)   0.00 (  0%)   0.01 (  2%)    1395 kB (  5%)
 parser inl. meth. body             :   0.02 (  9%)   0.06 ( 19%)   0.11 ( 19%)    2446 kB (  8%)
 template instantiation             :   0.05 ( 23%)   0.11 ( 35%)   0.17 ( 29%)    9572 kB ( 31%)
 integrated RA                      :   0.01 (  5%)   0.00 (  0%)   0.00 (  0%)      72 kB (  0%)
 thread pro- & epilogue             :   0.00 (  0%)   0.00 (  0%)   0.01 (  2%)       4 kB (  0%)
 TOTAL                              :   0.22          0.31          0.59          30848 kB
$ ls
hello.cpp  hello.preprocessed.cpp  hello.preprocessed.s
```

可以看到 `cc1plus` 输出了编译后的汇编文件 `hello.preprocessed.s`。可以打开来查看其中的内容。

???note "扩展知识 - C++ 命名混淆（Mangling）"

    在汇编语言文件中，你可能会看到类似 `_ZSt4endlIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_` 的奇怪符号。这是 C++ 编译器对函数进行命名混淆后的结果。C++ 中有函数重载，以及命名空间等 C 所没有的语言特性。为了实现这些特性，C++ 编译器会将函数的命名空间，以及调用参数的类型都编码到函数符号中，也就是相当于为每一个函数都创建了新的名字。这样就能区分开不同命名空间以及不同函数重载的同名函数了。

    需要注意的是，如何混淆一个函数的名称完全取决于编译器，C++ 标准中并没有规定编译器应当如何混淆函数名称。编译器只需要保证能够区分同名函数的不同重载即可。C++ 只规定了不允许只通过不同的返回值类型来重载函数，所以编译器**可能**会在混淆的时候省略返回值类型，也有可能不省略。

    系统中也自带了 `c++filt` 这个工具来帮助我们还原混淆符号对应的函数签名。我们运行 `c++filt` 后，输入混淆后的名称并回车，就能得到原始函数签名了。

    ```bash
    $ c++filt
    _ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc
    std::basic_ostream<char, std::char_traits<char> >& std::operator<< <std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&, char const*)
    _ZNSt8ios_base4InitC1Ev
    std::ios_base::Init::Init()
    ```

    可以看到编译器有时候会省略返回值类型有时候不会。
## 汇编

## 链接