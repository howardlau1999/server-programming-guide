# 回调模型

有时候我们希望开一个线程去执行一些耗时的操作，例如执行一个很复杂的计算，或者发送一个网络请求，并且想得到函数执行的结果。一种办法是使用一个回调函数接受结果。例如：

```cpp
#include <iostream>
#include <chrono>
#include <thread>
#include <functional>

int calculate(int arg) {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    return arg + 42;
}

void do_work(int arg, std::function<void(int)> callback) {    
    int result = calculate(arg);
    callback(result);
}

int main() {
    std::thread t(do_work, 1, [](int result) {
        std::cout << result << std::endl;
    });
    std::cout << "Continue main" << std::endl;
    t.join();
    return 0;
}
```

对于普通的函数，我们只需要用一个包装函数来负责将结果传递给回调函数，然后开启一个新的线程去调用这个包装函数即可。不过，回调函数容易陷入回调地狱中，当执行的操作一多起来的时候，就会产生非常深的回调嵌套。