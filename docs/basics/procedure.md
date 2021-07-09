# 从源代码到二进制

首先来看一个简单的程序，`hello.cpp`

```cpp
#include <iostream>
using namespace std;

int main() {
    cout << "Hello World!" << endl;
    return 0;
}
```

我们都知道，我们只需要运行 `g++ hello.cpp` 就可以将这个 C++ 源代码文件编译为可执行文件 `a.out` 了。但是在这个简单的命令背后，其实发生了许多事情。

## 预处理

