# systemd 部署

目前，较新的 Linux 发行版，都已经使用了 systemd 作为系统守护进程，负责系统各个服务进程的管理。我们可以使用 systemd 来守护我们的服务进程，实现自动重启等功能。

## 常用的目录安排

Linux 没有规定程序以及配置必须安装在哪个目录下，但为了管理方便，一般而言可以采用以下的存放路径：

### 可执行文件

全局的可执行文件一般安装在 `/usr/local` 下，其中可执行程序在 `/usr/local/bin` 下，而运行库等在 `/usr/local/lib` 下。一般这种程序是通过 `make install` 命令安装的。有一些只有一个可执行文件的命令行程序，也可以放在 `/usr/local/bin` 下。

对于已经提供了安装器的程序，例如 Matlab，则可以安装到 `/opt` 目录下，例如 Matlab 可以安装到 `/opt/matlab` 下，可执行文件一般会存放在 `/opt/xxx/bin/xxx` 中。

### 配置文件

配置文件一般有全局的配置文件，也有用户的配置文件。通常而言，全局配置文件由系统管理员管理，存放在 `/etc` 目录下。例如，`nginx` 程序的配置文件就存放在 `/etc/nginx` 目录下。而用户特定的配置文件，一般可以存放在 `~/.config` 目录下，或者 `~/.$APP` 目录下，例如，如果你写了一个服务器程序叫 `my-server`，可以将配置文件放到 `~/.my-server` 中。

不同的程序对于配置文件的搜索过程不一样，有的程序会内置若干个搜索路径，并按照优先级依次覆盖配置，有的程序必须手动指定一个配置文件路径，或者使用默认的配置文件路径启动。

## 编写 Unit 文件

通过 systemd 服务，我们可以让程序开机自启，并在崩溃退出的时候由系统自动重新启动。一般来说，我们需要在配置文件中指定：

- 服务的信息
- 服务的依赖
- 启停命令
- 运行环境

要创建一个 Unit，我们可以使用命令 `systemctl edit --full --force --user my-server.service` 来创建并编辑一个 Unit 文件。

在编辑器中，按照以下模板编写内容：

=== "my-server.service"
    ```ini
    [Unit]
    Description=my-server

    [Service]
    WorkingDirectory=/home/my-server
    Type=simple
    Environment=LOGLEVEL=info
    Environment=SECRET_KEY=secret
    ExecStart=/home/my-server/bin/my-server --port=8080
    Restart=on-failure
    User=my-server
    Group=my-server
    LimitCORE=infinity

    [Install]
    WantedBy=multi-user.target
    ```

其中 `ExecStart` 表示程序启动的命令，`WorkingDirectory` 表示程序运行的时候的工作目录，而 `Environment` 则表示程序运行时的环境变量。

保存退出编辑器之后，systemd 就会将文件内容放到合适的地方。这时候还不能自动启动程序，还需要执行 `systemctl --user enable my-server` 将程序加入到开机启动的列表中，然后执行 `systemctl --user start my-server` 将程序启动。我们还可以通过 `systemctl --user status my-server` 来查看程序运行的状态，通过 `journalctl --user -u my-server` 可以看到程序的标准输出内容。