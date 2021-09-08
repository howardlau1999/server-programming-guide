# Docker

Docker 是一套利用了 Linux 容器技术的工具。使用 Docker，我们可以很方便的将我们的应用程序打包，并在与系统隔离的环境中运行，就像是虚拟机一样。不过，在使用 Docker 前，需要安装 Docker。需要注意的是，Docker 只能在 Linux 下运行，如果你使用的是 Windows，需要借助 WSL2 或者虚拟机来运行。

???+note "cgroups 与 Linux Namespace"
    Docker 主要使用了 [cgroups](https://en.wikipedia.org/wiki/Cgroups) 技术以及 [Linux Namespace](https://en.wikipedia.org/wiki/Linux_namespaces) 技术。cgroups 用于资源的共享限制，例如限制 CPU 占用率、内存使用空间、磁盘 IO 等，而 Linux Namespace 则用于系统权限的隔离，例如隔离 IPC 空间、网络空间、用户权限空间等。

???note "OCI 标准"
    Docker 和 Linux 基金会提出了 [OCI 标准](https://opencontainers.org/)，促进容器技术的开放发展。理论上支持 OCI 标准的工具都可以互相替换，例如 [Podman](https://podman.io/) 等。

## 构建应用镜像

## 使用 Docker Compose 编排应用

