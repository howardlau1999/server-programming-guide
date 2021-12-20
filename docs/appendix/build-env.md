# 编译环境准备

在不同的操作系统上，最好还是使用操作系统厂商开发的编译工具链。例如，尽管我们可以通过 `mingw` 等工具在 Windows 上使用 `g++` 命令，但为了最好的兼容性，还是选择 `MSVC` 系列比较好。

鉴于每个人喜欢的操作系统不同，使用 Windows 编程也并没有什么问题。在编写一些操作系统无关的程序的时候，可以在自己更熟悉的环境下编程。

## Windows

在 Windows 上，我们一般会安装（已经过时的）DevC++，或者大名鼎鼎的 Visual Studio 这些 IDE。它们不仅能提供强大的编程环境，同时也会帮我们安装好编译器。不过，其实我们也可以在 Windows 上单独安装编译器，然后使用 Visual Studio Code 等编辑器来编程。

### 安装 Microsoft Build Tools

在 Windows 上，并不一定需要安装 Visual Studio 才能编译程序，可以安装 [Microsoft Build Tools](https://visualstudio.microsoft.com/thank-you-downloading-visual-studio/?sku=BuildTools&rel=16)，里面就包含了 MSVC 编译器。

点击上面的链接下载安装器之后，打开安装器，就可以看到安装界面：

![](img/ms-build-tools.png)

按照默认的选项安装桌面 C++ 开发工具即可。

### 配置 VSCode

安装完 Build Tools 之后，就可以按照[官方教程来配置 VSCode 使用 MSVC 编译器](https://code.visualstudio.com/docs/cpp/config-msvc)了。

## Linux

## WSL

WSL (Windows Subsystem for Linux) 是一个在 Windows 上能够运行原生 Linux 二进制可执行文件（ELF 格式）的兼容层。它是由微软与 Canonical 公司合作开发，其目标是使纯正的 Ubuntu、Debian 等映像能下载和解压到用户的本地计算机，并且映像内的工具和实用工具能在此子系统上原生运行。

可以在 powershell 中依次运行下述指令，安装 WSL1 版本的 Debian。

```powershell
wsl --set-default-version 1
wsl --install --distribution Debian
```

关于 WSL 的使用，最佳的教材即为[官方文档](https://docs.microsoft.com/zh-cn/windows/wsl/)，另外很推荐配合 VSCode 的[Remote-WSL](https://docs.microsoft.com/zh-cn/windows/wsl/tutorials/wsl-vscode)插件。假设你已经看过官方文档，以下内容给出一些日常使用经验。

WSL 的大部分功能都可以被完整 Linux 或装在虚拟机中的 Linux 代替，反之也一样。因此上述 Linux 的使用技巧均可使用在 WSL 中。

WSL 还存在一些局限性，例如 WSL1 不支持 GPU 和很多系统调用，WSL2 访问 Windows 文件性能差，两者（目前为止）均不能访问 USB 设备等。那除了日常使用更方便之外，WSL 还有其它的必要性吗？答案自然是有。当你的项目同时涉及到 Windows 和 Linux 环境下的混合编译时，WSL 一定是你的最佳选择；又例如，你可能会经常搞坏 Linux 环境，每次都要重装系统（无论物理机还是虚拟机）都会极大的消耗你的精力，而 WSL 不过是点一下卸载再重新点一下安装的事，前后几乎只有几分钟，并且还提供了打包备份的功能（`wsl --export`和 `wsl --import`）。

### WSL 访问 Windows

WSL 内可以通过 `powershell.exe` 调用 Windows 下的终端。

```bash
$ powershell.exe ls -File


    目录: D:\server-programming-guide


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----        2021/12/21      5:41          20138 LICENSE
-a----        2021/12/21      5:41           6505 mkdocs.yml
-a----        2021/12/21      5:41           1410 README.md
-a----        2021/12/21      5:41            156 requirements.txt
```

由于当前 WSL1 和 WSL2 都不支持很多操作（例如访问 USB 设备）。那么也可以借助 `powershell.exe` 去间接访问挂载在 Windows 环境中的软件环境和硬件外设。

### Windows 访问 WSL

同理，Windows 下也可以通过 wsl 调用 Linux 下的工具链。

```powershell
PS D:\server-programming-guide> wsl ls -l
total 32
drwxrwxrwx 1 wuk wuk  4096 Dec 21 05:41 docs
-rwxrwxrwx 1 wuk wuk 20138 Dec 21 05:41 LICENSE
-rwxrwxrwx 1 wuk wuk  6505 Dec 21 05:41 mkdocs.yml
-rwxrwxrwx 1 wuk wuk  1410 Dec 21 05:41 README.md
-rwxrwxrwx 1 wuk wuk   156 Dec 21 05:41 requirements.txt
```

例如，你的 Windows 没有配置 `git`，又需要临时下载一个项目，你可以像这样使用 WSL 中的 git 下载文件，无需单独开一个 WSL 终端。

```powershell
wsl git clone https://github.com/howardlau1999/server-programming-guide.git
```

当然要注意的是，如果项目中没有特别约束，那么这样做的一切设置（例如行末回车是`CRLF`还是`LF`）都是按照 Linux 中的设置来的，因此如果在 Windows 中直接打开的话可能会出现乱码的情况，此时换一个可以正确处理的编辑器（如 vscode）可解决。

## macOS
