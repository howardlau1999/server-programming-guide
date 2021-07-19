# 使用 CMake 编译多文件项目

本节将带你快速了解如何使用 CMake 来编译带有库的二进制可执行文件。

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

    ```cmake hl_lines="4-5"
    cmake_minimum_required(VERSION 3.16)
    project(my-server)

    include_directories("${CMAKE_CURRENT_SOURCE_DIR}/include")
    include_directories("${CMAKE_CURRENT_BINARY_DIR}")
    ```

## 生成库文件

接下来我们可以开始添加源代码文件，使用 `add_library` 来生成主程序所需要的库文件了。其中第一个参数是我们希望生成的库的名字，后续是生成这个库所需要的所有源文件。

=== "CMakeLists.txt"

    ```cmake hl_lines="7"
    cmake_minimum_required(VERSION 3.16)
    project(my-server)

    include_directories("${CMAKE_CURRENT_SOURCE_DIR}/include")
    include_directories("${CMAKE_CURRENT_BINARY_DIR}")

    add_library(mylib src/add.cpp src/sub.cpp)
    ```

这里我们使用了手工指定的方式，在文件比较多的时候可能会比较麻烦。不过，在依赖方面，最好是显式地指定所有依赖文件，一是方便工具分析，二是方便接手的人快速了解项目的依赖。

## 生成可执行二进制

最后，我们需要通过 `add_executable` 来生成可执行文件，并使用 `target_link_libraries` 将这个可执行文件链接到我们的库。

=== "CMakeLists.txt"

    ```cmake hl_lines="9-10"
    cmake_minimum_required(VERSION 3.16)
    project(my-server)

    include_directories("${CMAKE_CURRENT_SOURCE_DIR}/include")
    include_directories("${CMAKE_CURRENT_BINARY_DIR}")

    add_library(mylib src/add.cpp src/sub.cpp)

    add_executable(main src/main.cpp)
    target_link_libraries(main mylib)
    ```