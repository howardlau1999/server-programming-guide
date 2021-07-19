# 在项目中使用 CMake 引入第三方库

我们需要根据项目提供的配置文件，来确定合适的引用方式。

## `.cmake` 文件

如果你是用系统自带的包管理器安装的开发库，那么这个库有可能已经提供了 `.cmake` 文件，里面内置了使用这个库所需要的命令。这时候可以简单地使用 `find_package` 来让 CMake 帮我们加载配置文件。例如：

```bash
$ sudo apt install libfmt-dev
```

```cmake
find_package(fmt)
target_link_libraries(my-server fmt)
```

## CMakeLists.txt

如果项目提供了 CMakeLists.txt，那么我们直接使用 `add_subdirectory`，然后使用对应的 target 名称即可。例如：

```cmake
add_subdirectory("${CMAKE_CURRENT_SOURCE_DIR}/third_party/cpr")
target_link_libraries(my-server cpr)
```

如果不想将代码下载到自己的代码库中，可以使用 `FetchContent` 在构建的时候再下载，例如：

```cmake
include(FetchContent)
FetchContent_Declare(cpr GIT_REPOSITORY https://github.com/whoshuu/cpr.git GIT_TAG c8d33915dbd88ad6c92b258869b03aba06587ff9)
FetchContent_MakeAvailable(cpr)
```

## pkg-config

有一些库提供了 `pkg-config` 文件，后缀名是 `.pc`，我们可以使用 `pkg-config` 命令来在编译的时候输出合适的编译参数。例如：

```bash
$ sudo apt install libmysqlclient-dev
$ pkg-config --libs --cflags mysqlclient
-I/usr/include/mysql -lmysqlclient
$ g++ main.cpp `pkg-config --libs --cflags mysqlclient`
```

在 CMake 中，可以通过 `PkgConfig` 这个包及其提供的 `pkg_check_modules` 命令方便地引用已经存在的 `pkg-config` 文件。例如：

```cmake
find_package(PkgConfig REQUIRED)
pkg_check_modules(MySQL REQUIRED IMPORTED_TARGET mysqlclient>=21.0)
target_link_libraries(my-server PkgConfig::MySQL)
```

其中 `pkg_check_modules` 第一个参数为引入后的变量前缀，这个例子里，会引入 `MySQL_LIBRARIES` 等变量。为了方便使用，还可以通过 `IMPORTED_TARGET` 来提供 `PkgConfig::MySQL` 这样的引入形式，最后便是指定包名以及包版本号。