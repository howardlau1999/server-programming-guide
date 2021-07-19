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