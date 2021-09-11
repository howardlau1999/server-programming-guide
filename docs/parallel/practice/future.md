# Future 模式

Future 是“未来”的意思，也就是说这个值不是马上就能获得，而是要在“未来”某个时候才能得到值。`std::future` 除了存储具体的值以外，还存储了这个值的状态，使用者既可以轮询 `std::future` 的状态，也可以使用阻塞调用来让 `std::future` 的值确定之后再返回。前面就相当于一个简单的 `while` 循环，而后者则可以理解为条件变量的 `wait`。

而在没有设置值的时候，调用 `future` 的 `get()` 会阻塞并挂起线程，一旦有值则唤醒线程并返回值。可以理解为：

```cpp
template<typename T>
class future<T> {
    std::mutex mu;
    std::condition_variable cv;
    State state;
    T value;
protected:
    void set(T const& v) {
        std::scoped_lock lk(mu);
        state = READY;
        value = v;
        cv.notify_all();
    }
public:
    T get() {
        std::unique_lock lk(mu);
        cv.wait(lk, []() { state == READY });
        return value;
    }
    
    bool try_get(T& v) {
        std::scoped_lock lk(mu);
        if (state == READY) {
            v = value;
            return true;
        }
        return false;
    }
}
```

## `#!cpp std::promise`

`std::future` 并不能直接设置值，而是需要其他的包装类来改变它的内部状态，持有 `std::future` 的线程只能从中获取值。例如：

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

可以看到，`std::promise` 的使用方法也不太直观，有点像[回调模式](callback.md)，但又不完全像，还需要通过一个 `std::future` 才能拿到返回值。通过文档可以看到，`future` 只有 `get_value()`，而 `promise` 只有 `set_value()` 操作。也就是说，给 `std::promise` 设置值之后，甚至不能直接从里面将值获取回来，而 `future` 只能通过其他类来设置值，作为使用者只能从中获取值。

这种设计的逻辑是，通过类型来区分值的提供者和使用者，提供者不应该使用值而使用者不应该设置值，并通过 API 保证共享状态不会被错误地修改。



使用 Promise 可以避免回调地狱，用更自然地方法编写程序。

## `#!cpp std::packaged_task`

如果每次想启动线程执行别的任务的时候都要像上面一样先包装函数然后创建 promise 再启动线程就显得太过麻烦。C++ 中提供了 `std::packaged_task` 来完成包装函数和 promise 的部分。


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

int main() {
    std::packaged_task<int(int)> task(calculate);
    std::future<int> fut = task.get_future();
    std::thread t(std::move(task), 1);
    t.detach();
    std::cout << "Continue main" << std::endl;
    std::cout << fut.get() << std::endl;
    return 0;
}
```

包装后的任务并不会自动执行，但执行之后会自动设置 future 的值，我们只需要在执行任务之前获得 future，然后使用这个 future 来获取返回值即可。

## `#!cpp std::async`

如果不想手动开线程去执行任务，可以使用 `#!cpp std::async` 来直接启动异步任务，只需要去获得 `future` 的值即可。

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

int main() {
    std::future<int> fut = std::async(calculate, 1);
    std::cout << "Continue main" << std::endl;
    std::cout << fut.get() << std::endl;
    return 0;
}
```

这种方式使用最简单，但是对于执行的细节控制比较少。我们可以编写自己的**线程池**来实现类似的 API，同时能够把控更多细节。