# 在项目中使用 CMake 引入第三方库

我们需要根据项目提供的配置文件，来确定合适的引用方式。

## `.cmake` 文件

如果你是用系统自带的包管理器安装的开发库，那么这个库有可能已经提供了 `.cmake` 文件，里面内置了使用这个库所需要的命令。这时候可以简单地使用 `find_package` 来让 CMake 帮我们加载配置文件。我们以 `fmt` 库为例：

```bash
$ sudo apt install libfmt-dev
```

```cmake hl_lines="1"
find_package(fmt)
target_link_libraries(my-server fmt)
```

## CMakeLists.txt

有一些开源项目可能提供了 `CMakeLists.txt`，或者我们想引用我们自己或者同事、同学写的 CMake 项目，这时候可以用 `add_subdirectory` 来添加依赖，根据第三方依赖不同的放置位置，引用的方式也不太一样。

### 依赖是项目的子文件夹

假如我们的项目位于 `my-server` 文件夹，如果第三方依赖放在了 `my-server/third_party`，是项目的子文件夹，那么我们直接使用 `add_subdirectory`，然后使用对应的 target 名称即可。例如，假如我们使用了一个叫 [`cpr`](https://github.com/whoshuu/cpr.git) 的 CMake 项目，放在了 `my-server/third_party/cpr` 下，我们可以这么引用：

```cmake hl_lines="1"
add_subdirectory("${CMAKE_CURRENT_SOURCE_DIR}/third_party/cpr")
target_link_libraries(my-server cpr)
```

### 依赖和项目放在同一个层级

假如我们编写了几个项目 `my-server`、`another-server`，如果都用到了同一个依赖，那么在每个项目里都放一份代码可能会浪费空间，而且更新依赖也需要分别更新，这时候我们希望将第三方依赖放到和他们同一个级别的 `third_party` 目录中，还是以 `cpr` 项目为例，假如我们把依赖放在了 `third_party/cpr` 下，那么在 `my-server` 中，我们需要这么写：

```cmake hl_lines="1"
add_subdirectory("${CMAKE_CURRENT_LIST_DIR}/../third_party/cpr" cpr.build)
target_link_libraries(my-server cpr)
```

???+note "Try it"
    尝试不指定 `add_subdirectory` 的第二个参数，看看 CMake 的报错。

和前面的差不多，我们同样通过 `add_subdirectory` 指定项目的路径，但是，由于现在包含的路径不是项目的子文件夹了，需要我们手动指定依赖编译存放的路径，也就是 `add_subdirectory` 的第二个参数。

### 构建时再下载

如果不想将代码下载到自己的代码库中，可以使用 `FetchContent` 在构建的时候再下载，例如：

```cmake hl_lines="1-3"
include(FetchContent)
FetchContent_Declare(cpr GIT_REPOSITORY https://github.com/whoshuu/cpr.git GIT_TAG c8d33915dbd88ad6c92b258869b03aba06587ff9)
FetchContent_MakeAvailable(cpr)

target_link_libraries(my-server cpr)
```

## pkg-config

有一些库提供了 `pkg-config` 文件，后缀名是 `.pc`，我们可以使用 `pkg-config` 命令来在编译的时候输出合适的编译参数。例如：

```bash
$ sudo apt install libmysqlclient-dev
$ pkg-config --libs --cflags mysqlclient
-I/usr/include/mysql -lmysqlclient
$ g++ main.cpp `pkg-config --libs --cflags mysqlclient`
```

在 CMake 中，可以通过 `PkgConfig` 这个包及其提供的 `pkg_check_modules` 命令方便地引用已经存在的 `pkg-config` 文件。以 `mysqlclient` 这个库为例，它没有提供 `.cmake` 文件，也没有 `CMakeLists.txt`，但是提供了 `pkg-config` 文件：

```cmake hl_lines="1-2"
find_package(PkgConfig REQUIRED)
pkg_check_modules(MySQL REQUIRED IMPORTED_TARGET mysqlclient>=21.0)
target_link_libraries(my-server PkgConfig::MySQL)
```

其中 `pkg_check_modules` 第一个参数为引入后的变量前缀，这个例子里，会引入 `MySQL_LIBRARIES` 等变量。为了方便使用，还可以通过 `IMPORTED_TARGET` 来提供 `PkgConfig::MySQL` 这样的引入形式，最后便是指定包名以及包版本号。