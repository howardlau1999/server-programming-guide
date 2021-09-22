# 使用 CMake 编译多文件项目

本节将带你快速了解如何使用 CMake 来编译带有库的二进制可执行文件。在阅读本节之前，请确保已经完全理解并掌握了[程序的编译链接过程](../basics/procedure.md)。

## 组织项目目录

一般而言，C++ 的项目会将 `.h` 等头文件存放到项目根目录里的 `include` 文件夹，而将 `.cpp` 文件存放到项目根目录里的 `src` 文件夹。这两个文件夹里，再根据不同的模块需要组织子文件夹。而测试则单独存放到 `test` 文件夹。

而我们会将程序的大部分功能编译为库，然后编写一个简单的主函数文件，来将这些库有机地组合在一起，并驱动程序的运行。这样做的好处就是方便编写不同的驱动程序以及测试程序去调用我们写的函数。在更大型的项目合作中，有利于其他项目开发者复用我们的代码。

主函数文件可以单独存放到一个 `cmd` 文件夹中，也可以在 `src` 文件夹中以 `main.cpp` 或者其他名字存放。

## 指定项目元信息

首先在项目根目录创建一个 `CMakeLists.txt` 文件，并指定编译项目所需要的 CMake 版本以及项目名。请记住，在任何时候都要保持指定版本号的好习惯。

=== "CMakeLists.txt"

    ```cmake hl_lines="1-2"
    cmake_minimum_required(VERSION 3.16)
    project(my-server)
    ```

## 添加包含文件

为了使得我们编写的 `.cpp` 源文件能够使用在不同目录里的包含文件，我们需要在 CMake 中通过 `include_directories` 指定包含目录。

=== "CMakeLists.txt"

    ```cmake hl_lines="4"
    cmake_minimum_required(VERSION 3.16)
    project(my-server)

    include_directories("${CMAKE_CURRENT_SOURCE_DIR}/include")
    ```

## 生成库文件

接下来我们可以开始添加源代码文件，使用 `add_library` 来生成主程序所需要的库文件了。其中第一个参数是我们希望生成的库的名字，后续是生成这个库所需要的所有源文件。

=== "CMakeLists.txt"

    ```cmake hl_lines="6"
    cmake_minimum_required(VERSION 3.16)
    project(my-server)

    include_directories("${CMAKE_CURRENT_SOURCE_DIR}/include")

    add_library(mylib src/add.cpp src/sub.cpp)
    ```

这里我们使用了手工指定的方式，在文件比较多的时候可能会比较麻烦。不过，在依赖方面，最好是显式地指定所有依赖文件，一是方便工具分析，二是方便接手的人快速了解项目的依赖。

## 生成可执行二进制

最后，我们需要通过 `add_executable` 来生成可执行文件，并使用 `target_link_libraries` 将这个可执行文件链接到我们的库。

=== "CMakeLists.txt"

    ```cmake hl_lines="8-9"
    cmake_minimum_required(VERSION 3.16)
    project(my-server)

    include_directories("${CMAKE_CURRENT_SOURCE_DIR}/include")

    add_library(mylib src/add.cpp src/sub.cpp)

    add_executable(main src/main.cpp)
    target_link_libraries(main mylib)
    ```

## 编译

通常，我们会新建一个另外的编译文件夹进行源代码外编译（Out-of-source build）。这样有个好处是编译的产物不会和源代码混在一起，如果编译出错了，我们直接把整个编译文件夹删掉重来即可。使用版本管理软件的时候也可以将整个文件夹排除掉。使用 `cmake` 命令就可以帮助我们生成构建系统所需要的项目文件了。

```bash
$ mkdir build && cd build
$ cmake ..
-- The C compiler identification is GNU 10.3.0
-- The CXX compiler identification is GNU 10.3.0
-- Check for working C compiler: /usr/bin/cc
-- Check for working C compiler: /usr/bin/cc -- works
# ...
-- Configuring done
-- Generating done
-- Build files have been written to: /home/howard/a/build
$ make       
make[1]: Entering directory '/home/howard/a/build'
make[2]: Entering directory '/home/howard/a/build'
Scanning dependencies of target mylib
make[2]: Leaving directory '/home/howard/a/build'
make[2]: Entering directory '/home/howard/a/build'
[ 20%] Building CXX object CMakeFiles/mylib.dir/src/add.cpp.o
[ 40%] Building CXX object CMakeFiles/mylib.dir/src/sub.cpp.o
[ 60%] Linking CXX static library libmylib.a
make[2]: Leaving directory '/home/howard/a/build'
[ 60%] Built target mylib
make[2]: Entering directory '/home/howard/a/build'
Scanning dependencies of target main
make[2]: Leaving directory '/home/howard/a/build'
make[2]: Entering directory '/home/howard/a/build'
[ 80%] Building CXX object CMakeFiles/main.dir/src/main.cpp.o
[100%] Linking CXX executable main
make[2]: Leaving directory '/home/howard/a/build'
[100%] Built target main
make[1]: Leaving directory '/home/howard/a/build'
$ ./main
add(1, 2) = 3
sub(1, 2) = -1
```

???+note "使用 `-G` 选项指定构建系统"
    CMake 支持生成不同的构建系统的文件，使用 `cmake -G` 可以查看可以使用哪些生成器。
    ```bash
    $ cmake -G
    CMake Error: No generator specified for -G

    Generators
        * Unix Makefiles             = Generates standard UNIX makefiles.
        Green Hills MULTI            = Generates Green Hills MULTI files
                                        (experimental, work-in-progress).
        Ninja                        = Generates build.ninja files.
        Watcom WMake                 = Generates Watcom WMake makefiles.
        CodeBlocks - Ninja           = Generates CodeBlocks project files.
        CodeBlocks - Unix Makefiles  = Generates CodeBlocks project files.
        CodeLite - Ninja             = Generates CodeLite project files.
        CodeLite - Unix Makefiles    = Generates CodeLite project files.
        Sublime Text 2 - Ninja       = Generates Sublime Text 2 project files.
        Sublime Text 2 - Unix Makefiles
                                     = Generates Sublime Text 2 project files.
        Kate - Ninja                 = Generates Kate project files.
        Kate - Unix Makefiles        = Generates Kate project files.
        Eclipse CDT4 - Ninja         = Generates Eclipse CDT 4.0 project files.
        Eclipse CDT4 - Unix Makefiles= Generates Eclipse CDT 4.0 project files.
    ```
    我们可以尝试使用 `ninja` 来编译项目：
    ```bash
    $ cd .. && rm -rf build
    $ mkdir build && cd build
    $ cmake .. -G"Ninja"
    -- The C compiler identification is GNU 10.3.0
    -- The CXX compiler identification is GNU 10.3.0
    -- Check for working C compiler: /usr/bin/cc
    -- Check for working C compiler: /usr/bin/cc -- works
    # ...
    -- Configuring done
    -- Generating done
    -- Build files have been written to: /home/howard/a/build    
    $ ninja
    [5/5] Linking CXX executable main
    $ ./main
    add(1, 2) = 3
    sub(1, 2) = -1
    ```

## 使用建议

CMake 其实存在很多设计上的缺陷，例如大量使用全局变量进行传参，还有混乱的语法，都是使用 CMake 的一道门槛，而且 CMake 的指令也很难记得住，它的“声明式命令”也是出了名的难调试。所以，建议在使用 CMake 的时候，先找几个容易理解的 CMake 项目，然后复制粘贴需要的部分，而不是每次都从头写。当然你也可以在这个过程中，整理自己常用的 CMake 代码片段，以后直接以此为模板进行修改即可。