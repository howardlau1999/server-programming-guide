# 概览

在服务器程序中，为了更充分地利用现代 CPU 的多核性能，一般会使用多线程编程的方式来使我们的程序尽可能多地处理并行请求。本节简单介绍 C++ 中常用的多线程编程库以及编程范式。

## 使用 `#!cpp std::thread` 启动多个线程

在 C++ 中，要启动多个线程，只需要使用 `#!cpp std::thread` 这个类。类的构造函数第一个参数为希望执行的函数，后面的参数则是需要传递给这个函数的参数。当 `#!cpp std::thread` 对象被创建的时候，线程就会开始执行。如果我们需要等待线程执行完毕才退出，那么需要调用线程对象的 `join()` 方法。

=== "mt1.cpp"

    ```cpp
    #include <thread>
    #include <iostream>
    
    void thread_function(int a) {
        std::cout << "a = " << a << std::endl;
    }

    int main() {
        std::thread t1(thread_function, 1); 
        std::thread t2(thread_function, 2); 

        t1.join();
        t2.join();

        return 0;
    }
    ```

如果我们不调用线程的 `join()` 方法，那么程序结束时将会出错。

```bash
$ g++ mt1.cpp -lpthread
$ ./a.out 
terminate called without an active exception
[1]    1245899 abort (core dumped)  ./a.out
```

如果我们希望启动线程后就不需要关心线程的状态，可以调用 `detach()` 方法将线程独立出去。

=== "mt1.cpp"

    ```cpp hl_lines="12 13"
    #include <thread>
    #include <iostream>
    
    void thread_function(int a) {
        std::cout << "a = " << a << std::endl;
    }

    int main() {
        std::thread t1(thread_function, 1); 
        std::thread t2(thread_function, 2); 

        t1.detach();
        t2.detach();

        return 0;
    }
    ```

???+info
    `#!cpp std::thread` 的构造函数默认使用拷贝传值。这意味着如果你的函数期望收到一个引用，简单地向线程构造函数传入变量是不行的，这种行为在现代 C++ 编译器中会被视为编译错误，就像下面的代码一样：

    === "mt.cpp"

        ```cpp hl_lines="11"
        #include <thread>
        #include <iostream>

        void thread_function(int &value) {
            // 期望收到一个引用，这样就能改变它的值
            value++;
        }

        int main() {
            int value = 0;
            std::thread t(thread_function, value); // 直接传入 value
            t.join();
            std::cout << value << std::endl;
            return 0;
        }
        ```

    根据 [cppreference.com](https://en.cppreference.com/w/cpp/thread/thread/thread#Notes)，我们需要在传入引用时使用 `#!cpp std::ref` 来包裹变量名，即将上面代码的 `#!cpp std::thread` 构造函数调用中的参数 `value` 改为 `!#cpp std::ref(value)`。

    当函数需要常量引用时，我们也可以使用 `#!cpp std::cref`。

## 线程间共享内存

在很多场景下，不同的线程之间需要传递信息以达成某种协作关系。

例如：在一个基于多线程技术的数据计算软件里，有一个线程负责读取用户输入的数据，还有若干个线程负责根据输入的数据执行相当费时的查询任务。在这一场景下，我们通常将前一个线程称为生产者线程，将后一个线程称为消费者线程。它们之间需要传递用户输入的数据来让整个软件的功能得以实现。

为了完成线程之间的交流，有一种办法是共享内存。它利用了一个事实：由同一个进程产生的不同线程，共享了原进程的内存空间。这意味着一个线程能够访问的变量，大部分也能够被另一个线程所访问。下面我们展示一段实现了上述软件功能的代码：

=== "bad_case.cpp"

    ```cpp
    int list[100];  // 用来存储用户输入的数据，每一份数据用 int 表示
    int index = -1; // 用 index 指向当前还没有被处理的数据

    void producer() {
        while (true) {
            // 这里忽略对 list 是否饱和的检查
            // 下面获取用户输入的数据并写入 list 中，如果用户
            // 没有输入那么线程就会卡在 read_data() 调用内部而不会返回
            list[index + 1] = read_data(); 
            index++;
        }
    }

    void consumer() {
        while (index >= 0) {
            int data = list[index--]; // 从 list 中拿到数据
            do_calculate(data);
        }
    }
    ```

上述代码看上去似乎可以正常工作，然而真正运行的时候我们很可能看到一些错误的现象，例如用户输入了一份数据，结果两个消费者线程都拿到了这个数据，重复地对数据进行了计算。又如，用户输入了新的一份数据之后，始终没有等到消费者线程的输出。还有可能我们的程序会在输入数据后直接崩溃（访问了无效的数组下标）。这段代码充满了问题！

### 竞态条件

导致上述现象的原因是：多个线程之间对 `list` 和 `index` 变量竞争访问，导致了竞态条件（race condition）的出现。以 `index` 为例，我们解释其中一个现象——重复运算的原因。

由于多个线程之间并发运行，它们中的代码的执行顺序是相互独立的。因此当生产者线程得到了用户的输入，将 `index` 置为 `1` 之后，假设有两个消费者线程 A 和 B，它们都尝试读取 `index` 的值并发现它已经大于等于 `0`，于是顺利地进入了循环内部。这种情况也有可能不会发生，当 A 线程动作快一些在完成了 `index--` 之后，B 线程就不会进入循环了。需要强调的是：**正如我们前面提到的，并发线程的代码执行顺序相互独立，因此我们不能对它们会以何种顺序访问共享变量 `index` 作出任何假设**。这就是竞态条件的本质，我们无法推测程序究竟会有哪些行为。

有兴趣的读者可以自行推导其他两种情景产生的原因，它们都是基于某种有问题的变量访问顺序。

???+info
    上述的例子准确来说应该叫做数据竞争（data race），它只是竞态条件的其中一种，却也是计算机软件里最常见的种类之一。如果你想了解更多关于竞态条件的信息，请使用搜索引擎。本文中的竞态条件指的都是数据竞争。

为了在共享内存时避免竞态条件导致的错误结果，人们发明了许多手段。其中最简单的手段是使用互斥量（mutex）保护存在竞态条件的变量。将上文的代码片段修改为如下所示就能解决竞态条件导致的问题：

=== "bad_case.cpp"

    ```cpp hl_lines="9 12 18 21"
    int list[100];  // 用来存储用户输入的数据，每一份数据用 int 表示
    int index = -1; // 用 index 指向当前还没有被处理的数据
    std::mutex m;

    void producer() {
        while (true) {
            // 把会阻塞线程运行的操作单独放出来
            int data = read_data();
            m.lock();
            list[index + 1] = data; 
            index++;
            m.unlock();
        }
    }

    void consumer() {
        while (true) {
            m.lock();
            if (index < 0) continue; // 没有新数据时直接跳过本次循环
            int data = list[index--];
            m.unlock();
            // 下面复杂的计算不需要访问 list 和 index
            // 因此不会出现竞态条件
            do_calculate(data);
        }
    }
    ```

对互斥量进行 `lock()` 操作是在尝试获得一个“独享权”，一旦当前线程获得了这个权利，其他线程在调用 `lock()` 时就无法获得这种权利，一直被阻塞无法进行下一步操作。直到拿到了权利的线程通过 `unlock()` 放弃权利之后，被阻塞的线程们又会进入下一轮争夺。由此，我们通过使用互斥量，在 `lock()` 和 `unlock()` 的代码之间确保了：**在任意时间点，最多只有一个线程会访问 `list` 和 `index`，竞态条件不复存在**。

所谓的“独享权”也被比喻为锁，这也是上述两个函数名字的由来。而且也由于上面这种常见的用法，互斥量也被称为互斥锁。

`lock()` 和 `unlock()` 包裹的这段代码被称为临界区（critical section），因为它们访问了共享变量。

???+info
    为什么需要把 `read_data()` 放到 `lock()` 前面？因为我们提到这个函数会阻塞当前函数的运行，直到用户提交新的数据。因此假如我们放在 `lock()` 之后，当用户迟迟不提交数据时，我们的消费者线程也不能够访问临界区，导致程序工作的低效率。

    同理，将 `do_calculate()` 放到 `unlock()` 之后也是为了尽可能缩短线程持有锁的时间，提高线程的并发效率。

???+info
    互斥锁会在没有拿到锁之后阻塞当前线程的运行，并让操作系统将 CPU 的执行切换到其他线程，这里会发生线程上下文切换，需要消耗很多 CPU 指令周期，有时候消耗的时间甚至比持有锁时修改变量的时间还长。
    
    为了解决这个问题，人们发明了另一种互斥量，这种互斥量在无法拿到锁时会进入类似 `while(true)` 的死循环，而不是休眠当前线程。这种互斥量也被称为“自旋锁”。