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