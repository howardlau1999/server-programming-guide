# 代码格式化

保持代码风格的统一以及整洁可以减少他人阅读代码的成本。代码格式化工具可以根据配置来对代码进行格式检查以及自动修正，例如缩进对齐、缩进用空格还是 Tab、代码换行的琐碎的细节。代码格式化工具的原理是将代码解析一遍，构造好语法树，然后按照指定格式将这个语法树重新输出出来。

使用代码格式化软件的时候，一般是将格式要求保存在一个配置文件中，并且将这个文件也保存到代码仓库里，这样其他人只需要下载相同的格式化工具运行，就能得到风格统一的代码了。有时候有些人可能没有运行格式化工具就提交了代码，也可以使用自动化工具来运行格式化检查，阻止这次提交。

不同语言有不同的代码格式化工具，这里以 C++ 使用的 [`clang-format`](https://clang.llvm.org/docs/ClangFormat.html) 为例。

???+note
    Ubuntu 上可以直接使用 `apt` 工具安装 `clang-format` 包。

## 编写配置文件

通常代码格式化工具都会提供数十上百的配置选项，如果我们一个个配置的话会比较麻烦，所以我们使用格式化工具的时候，一般是直接使用一个看上去比较适合需求的配置，然后再对其做略微修改。例如，`clang-format` 内置了 `LLVM`、`Google`、`Microsoft` 等常用的配置，可以使用命令行将其写到文件中：

```bash
$ clang-format -style=llvm -dump-config > .clang-format
```

???+note
    不同的代码格式化工具的配置文件名称不一样，具体可以参考对应的工具文档。同时，目前的 IDE 或者文本编辑器插件，也支持读取配置文件并运行格式化工具。

使用文本编辑器打开 `.clang-format` 文件，可以看到这是一个 YAML 格式的文件：

```yaml
---
Language:        Cpp
# BasedOnStyle:  LLVM
AccessModifierOffset: -2
AlignAfterOpenBracket: Align
AlignConsecutiveMacros: false
AlignConsecutiveAssignments: false
AlignConsecutiveDeclarations: false
AlignEscapedNewlines: Right
AlignOperands:   true
AlignTrailingComments: true
AllowAllArgumentsOnNextLine: true
AllowAllConstructorInitializersOnNextLine: true
# ...
```

具体的配置选项，可以参考 [`clang-format` 的官方文档](https://clang.llvm.org/docs/ClangFormat.html)。

## 运行代码格式化工具

下面以一个格式混乱的代码作为例子，演示下代码格式化工具的作用：

=== "hello.cpp"

    ```cpp
    #include<iostream>
    #include <string>
    class Greeter{
    public:
     Greeter():message("hello, world"){};
            void SayHello(){std::cout<<message<<std::endl;}private:
      std::string message;
    };int main(){Greeter().SayHello();return 0;}
    ```

运行 `clang-format`，可以得到格式整洁的代码输出：

```text
$ clang-format hello.cpp
#include <iostream>
#include <string>
class Greeter {
public:
  Greeter() : message("hello, world"){};
  void SayHello() { std::cout << message << std::endl; }
private:
  std::string message;
};
int main() {
  Greeter().SayHello();
  return 0;
}
```

如果希望格式化工具直接修改文件，而不是输出到标准输出，通常工具也会提供对应的参数来原地修改文件，例如，`clang-format` 提供了 `-i` 选项：

```text
$ clang-format hello.cpp
$ cat hello.cpp
#include <iostream>
#include <string>
class Greeter {
public:
  Greeter() : message("hello, world"){};
  void SayHello() { std::cout << message << std::endl; }
private:
  std::string message;
};
int main() {
  Greeter().SayHello();
  return 0;
}
```

不过，`clang-format` 并没有提供一次格式化目录下所有文件的方法，需要我们自行编写脚本来批量格式化代码。例如，我们可以使用这样的命令，每次提交代码前运行一次：

```bash
$ find my-server/ -iname *.h -o -iname *.cpp | xargs clang-format -i
```

## 配置编辑器

当然如果每次都手动执行命令就太麻烦了，所以我们还需要配置文本编辑器来自动格式化我们的代码，在 Visual Studio Code 中，我们可以使用 [clang-format 插件](https://marketplace.visualstudio.com/items?itemName=xaver.clang-format)，在 CLion 里，则已经内置了支持。

为了避免忘记格式化代码，我们还可以配置编辑器中类似 `Format On Save` 等选项，在保存文件的时候就运行一次格式化工具。