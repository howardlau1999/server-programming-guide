# Promise 模式

Promise 是“承诺”的意思，看上去并不是很好理解，它的作用是将一个值存放起来，并提供给另一个线程读取。举个例子：

```cpp
#include <iostream>
#include <chrono>
#include <thread>
#include <functional>
#include <future>

int calculate(int arg) {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    return arg + 42;
}

void do_work(int arg, std::promise<int> p) {    
    int result = calculate(arg);
    p.set_value(result);
}

int main() {
    std::promise<int> p;
    std::future<int> fut = p.get_future();
    std::thread t(do_work, 1, std::move(p));
    t.detach();
    std::cout << "Continue main" << std::endl;
    std::cout << fut.get() << std::endl;
    return 0;
}
```

可以看到，promise 的使用方法也不太直观，有点像[回调模式](callback.md)，但又不完全像，还需要通过一个 `future` 才能拿到返回值。`future` 又是什么？为什么不能直接从 `promise` 取值？通过文档可以看到，`future` 只有 `get_value()`，而 `promise` 只有 `set_value()` 操作。也就是说，给 `promise` 设置值之后，甚至不能直接从里面将值获取回来，而 `future` 只能通过其他类来设置值，作为使用者只能从中获取值。

这种设计的逻辑是，通过类型来区分值的提供者和使用者，提供者不应该使用值而使用者不应该设置值，并通过 API 保证共享状态不会被错误地修改。

而在没有设置值的时候，调用 `future` 的 `get()` 会阻塞并挂起线程，一旦有值则唤醒线程并返回值。可以理解为：

```cpp
cv.wait(lk, []() { state == READY });
return value;
```

使用 Promise 可以避免回调地狱，用更自然地方法编写程序。