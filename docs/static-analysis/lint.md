# 静态检查

除了代码的格式问题以外，静态检查还可以检查出一些逻辑错误。例如，忘记在基类的虚构函数添加 `#!cpp virtual` 关键词，又或者是使用了 `#!cpp using namespace std` 这种会污染命名空间的指令。这一步通常称为 lint。常用的 C++ 静态检查工具有谷歌开发的 `cpplint.py` 以及 `clang-tidy`，其中 `cpplint.py` 是基于纯文本分析，而且只检查 Google C++ Style Guide 中的规则。而 `clang-tidy` 则会调用 `clang` 工具链对代码进行解析等，支持更多内置规则，功能上会更强大。

## `cpplint.py`

使用 `cpplint.py` 前需要安装 Python 3 以及 pip 工具，然后使用 pip 工具安装即可。

```bash
$ sudo apt install python3 python3-pip
$ pip install --user cpplint
```

使用方法也比较简单，直接 `cpplint <文件名>` 即可。例如，我们有这样的代码：

=== "src/main.cpp"

    ```cpp
    #include <iostream>
    #include "mylib.h"
    using namespace std;

    int main() {
        cout << "add(1, 2) = " << add(1, 2) << endl;
        cout << "sub(1, 2) = " << sub(1, 2) << endl;
        return 0;
    }
    ```

运行 `cpplint` 工具，可以得到这样的输出：

```bash
$ cpplint src/main.cpp
src/main.cpp:0:  No copyright message found.  You should have a line: "Copyright [year] <Copyright Owner>"  [legal/copyright] [5]
src/main.cpp:2:  Include the directory when naming .h files  [build/include_subdir] [4]
src/main.cpp:3:  Do not use namespace using-directives.  Use using-declarations instead.  [build/namespaces] [5]
Done processing src/main.cpp
Total errors found: 3
```

可以看到短短的一段代码就有三个不规范的地方，当然，我们也可以根据自己需要，去除不需要的规则，比如我们选择不遵守关于 copyright 的规则：

```bash
$ cpplint --filter=-legal/copyright src/main.cpp
src/main.cpp:2:  Include the directory when naming .h files  [build/include_subdir] [4]
src/main.cpp:3:  Do not use namespace using-directives.  Use using-declarations instead.  [build/namespaces] [5]
Done processing src/main.cpp
Total errors found: 2
```

那么最后的错误报告就不会报告关于 copyright 的规则。

## `clang-tidy`

[`clang-tidy`](https://clang.llvm.org/extra/clang-tidy/) 是基于 clang 工具链的静态分析工具，比起 `cpplint.py` 功能更全面，内置的规则也更多，检查也更准确。不过因为是基于代码分析的，所以需要提供编译指令信息，在代码可以正常编译的情况下才能进行分析，而且占用资源更多，速度比起文本分析的工具也更慢。

可以直接使用 `apt` 包管理器安装 `clang-tidy`：

```bash
$ sudo apt install clang-tidy
```

=== "src/main.cpp"

    ```cpp
    #include <iostream>
    #include "mylib.h"
    using namespace std;

    int main() {
        cout << "add(1, 2) = " << add(1, 2) << endl;
        cout << "sub(1, 2) = " << sub(1, 2) << endl;
        return 0;
    }
    ```

=== "include/mylib.h"

    ```cpp
    int add(int a, int b);
    int sub(int a, int b);
    ```

如果我们直接运行 `clang-tidy` 则会得到报错：

```bash
$ clang-tidy -checks=google* src/main.cpp
Error while trying to load a compilation database:
Could not auto-detect compilation database for file "src/main.cpp"
No compilation database found in /home/howard/lint/src or any parent directory
fixed-compilation-database: Error while opening fixed database: No such file or directory
json-compilation-database: Error while opening JSON database: No such file or directory
Running without flags.
1 error generated.
Error while processing /home/howard/lint/src/main.cpp.
/home/howard/lint/src/main.cpp:2:10: error: 'mylib.h' file not found [clang-diagnostic-error]
#include "mylib.h"
         ^
Found compiler error(s).
```

和 `cpplint.py` 不同，`clang-tidy` 需要尝试编译文件，而 `mylib.h` 不在同一个文件夹，它找不到，所以报错了。我们需要提供编译的一些选项给 `clang-tidy`。

一种方式是错误中提到的 **Compilation Database**，一般保存在 `compile_commands.json` 文件中，里面保存了针对每个文件的编译指令。这个文件可以通过将 CMake 的 `CMAKE_EXPORT_COMPILE_COMMANDS` 选项设为 `1` 生成。

当然，也可以直接在命令行指定编译选项：

```bash
$ clang-tidy -checks=google* src/main.cpp -- -Iinclude
1205 warnings generated.
/home/howard/lint/src/main.cpp:3:1: warning: do not use namespace using-directives; use using-declarations instead [google-build-using-namespace]
using namespace std;
^
```

和 `clang-format` 类似，`clang-tidy` 支持把配置以 yaml 格式保存到文件中，名称是 `.clang-tidy`。

```yaml
---
Checks:     '
            bugprone-*,
            clang-analyzer-*,
            google-*,
            modernize-*,
            performance-*,
            portability-*,
            readability-*,
            -bugprone-too-small-loop-variable,
            -clang-analyzer-cplusplus.NewDelete,
            -clang-analyzer-cplusplus.NewDeleteLeaks,
            -modernize-use-nodiscard,
            -modernize-avoid-c-arrays,
            -readability-magic-numbers,
            -bugprone-branch-clone,
            -bugprone-signed-char-misuse,
            -bugprone-unhandled-self-assignment,
            -clang-diagnostic-implicit-int-float-conversion,
            -modernize-use-auto,
            -modernize-use-trailing-return-type,
            -readability-convert-member-functions-to-static,
            -readability-make-member-function-const,
            -readability-qualified-auto,
            -readability-redundant-access-specifiers,
            '
CheckOptions:
 - { key: readability-identifier-naming.ClassCase,           value: CamelCase  }
 - { key: readability-identifier-naming.EnumCase,            value: CamelCase  }
 - { key: readability-identifier-naming.FunctionCase,        value: CamelCase  }
 - { key: readability-identifier-naming.GlobalConstantCase,  value: UPPER_CASE }
 - { key: readability-identifier-naming.MemberCase,          value: lower_case }
WarningsAsErrors: '*'
HeaderFilterRegex: '/(src|test)/include'
AnalyzeTemporaryDtors: true
```