# XMake

[XMake](https://xmake.io/) 是基于 Lua 语言的编译系统，它结合了包管理和编译系统，只需要编写一个 xmake 脚本，就能自动下载依赖并编译项目，不需要像 CMake 那样需要手动处理依赖的获取，也不需要像 conan 以及 vcpkg 等第三方包管理软件那样需要手动修改 CMake 脚本。同时 XMake 内置了除了 Windows 以及 Linux 之外许多平台的支持，例如 Android、iOS 等，也可以兼容已有的 CMake/Makefile 等项目。