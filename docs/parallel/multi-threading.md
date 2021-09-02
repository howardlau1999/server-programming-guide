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

    根据 [cppreference.com](https://en.cppreference.com/w/cpp/thread/thread/thread#Notes)，我们需要在传入引用时使用 `std::ref` 来包裹变量名，即将上面代码的 `#!cpp std::thread` 构造函数调用中的参数 `value` 改为 `std::ref(value)`。

    当函数需要常量引用时，我们也可以使用 `std::cref`。

## 线程间同步

在很多场景下，不同的线程之间需要传递信息以达成某种协作关系。

例如：在一个基于多线程技术的数据计算软件里，有一个线程负责读取用户输入的数据，还有若干个线程负责根据输入的数据执行相当费时的查询任务。在这一场景下，我们通常将前一个线程称为生产者线程，将后一个线程称为消费者线程。它们之间需要传递用户输入的数据来让整个软件的功能得以实现。

为了完成线程之间的交流，有一种办法是共享内存。它利用了一个事实：**由同一个进程产生的不同线程，共享了原进程的内存空间。**这意味着一个线程能够访问的变量，大部分也能够被另一个线程所访问。下面我们展示一段实现了上述软件功能的代码：

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

导致上述现象的原因是：多个线程对 `index` 变量竞争访问与修改，导致了**竞态条件（race condition）**的产生。简单来说，竞态条件的发生是因为：**某种操作的结果依赖于两个或多个线程进行的操作的相对顺序**。

我们解释其中一个现象——重复运算的原因。由于多个线程之间并发运行，它们中的代码的执行顺序是相互独立的。因此当生产者线程得到了用户的输入，将 `index` 置为 `1` 之后，假设有两个消费者线程 A 和 B，它们都尝试读取 `index` 的值并发现它已经大于等于 `0`，于是顺利地进入了循环内部。这种情况也有可能不会发生，当 A 线程动作快一些在完成了 `index--` 之后，B 线程就不会进入循环了。

正如我们前面提到的，**并发线程的代码执行顺序相互独立，因此我们不能对它们会以何种顺序访问共享变量 `index` 作出任何假设**，这和竞态条件发生的原因相呼应。有兴趣的读者可以自行推导其他两种情景产生的原因，它们都是基于某种有问题的变量访问顺序。

???+info
    上述的例子准确来说应该叫做**数据竞争（data race）**，它只是竞态条件的其中一种，却也是计算机软件里最常见的种类之一。本文中的竞态条件指的都是数据竞争。

### 互斥量

为了在共享内存时避免竞态条件导致的错误结果，人们发明了许多手段。其中最简单的手段是使用互斥量（mutex）保护存在竞态条件的变量。将上文的代码片段修改为如下所示就能解决竞态条件导致的问题：

=== "with_mutex.cpp"

    ```cpp hl_lines="9 12 18 20 24"
    int list[100];  // 用来存储用户输入的数据，每一份数据用 `int` 表示
    int index = -1; // 用 `index` 指向当前还没有被处理的数据
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
            if (index < 0) {
                m.unlock(); // 在使用 `continue` 和 `break` 语句时非常容易忘记进行 `unlock()`
                continue;   // 没有新数据时直接跳过本次循环
            }
            int data = list[index--];
            m.unlock();
            // 下面复杂的计算不需要访问 `index`
            // 因此不会出现竞态条件
            do_calculate(data);
        }
    }
    ```

对互斥量进行 `lock()` 操作是在尝试获得一个“独享权”，一旦当前线程获得了这个权利，其他线程在调用 `lock()` 时就无法获得这种权利，一直被阻塞无法进行下一步操作。直到拿到了权利的线程通过 `unlock()` 放弃权利之后，被阻塞的线程们又会进入下一轮争夺。由此，我们通过使用互斥量，在 `lock()` 和 `unlock()` 的代码之间确保了：**在任意时间点，最多只有一个线程会访问 `list` 和 `index`，竞态条件不复存在**。

所谓的“独享权”也被比喻为锁，就好像上锁之后别人就无法进门，直到自己解锁才行，这也是上述两个函数名字的由来。而且也由于上面这种常见的用法，互斥量也被称为互斥锁。另外，`lock()` 和 `unlock()` 包裹的这段代码被称为**临界区（critical section）**，因为它们访问了相同的共享变量。

???+info
    为什么需要把 `read_data()` 放到 `lock()` 前面？因为我们提到这个函数会阻塞当前函数的运行，直到用户提交新的数据。因此假如我们放在 `lock()` 之后，当用户迟迟不提交数据时，我们的消费者线程也不能够访问临界区，导致程序工作的低效率。

    同理，将 `do_calculate()` 放到 `unlock()` 之后也是为了尽可能缩短线程持有锁的时间，提高线程的并发效率。

???+info
    互斥锁会在没有拿到锁之后阻塞当前线程的运行，并让操作系统将 CPU 的执行切换到其他线程，这里会发生线程上下文切换，需要消耗很多 CPU 指令周期，有时候消耗的时间甚至比持有锁时修改变量的时间还长。
    
    为了解决这个问题，人们发明了另一种互斥量，这种互斥量在无法拿到锁时会进入类似 `while(true)` 的死循环，而不是休眠当前线程。这种互斥量也被称为“自旋锁”。

???+warning
    千万不要忘记调用 `unlock()`！特别是当我们在循环中使用了 `continue` 或者 `break` 时，我们很容易忘记在其之前进行 `unlock()`。
    
    假如我们在上面的 `consumer()` 函数里漏掉了 `m.unlock()`，那么我们会在之后的循环里又一次调用 `m.lock()`，**这对 C++ 来说是未定义行为（undefined behavior），会导致一些无法预料的负面后果，我们的程序可能会直接崩溃。**
    
    为此，我们建议使用 `std::scoped_lock`，它在构造函数中自动调用 `lock()`，并在析构函数中自动调用 `unlock()`，是 C++ 的 RAII 思想的实践之一。我们可以把 `consumer()` 的代码修改为如下所示：

    ```cpp hl_lines="5"
    void consumer() {
        int data = -1;
        while (true) {
            {
                std::scoped_lock lock(m); // 构造函数自动调用 `m.lock()`
                if (index < 0) continue;  // 离开本次循环时，`lock` 的析构函数会被执行
                data = list[index--];
            } // 离开本代码块时，`lock` 的析构函数也会被执行
            do_calculate(data);
        }
    }
    ```

    **使用 RAII，特别是带有默认构造函数的类时，务必需要注意是否给变量起了名字。**例如：

    ```cpp hl_lines="2"
    void func() {
        std::scoped_lock(m);
    }
    ```

    如果没有起名字，那么这个语句可以有两种理解：一是调用了 `std::scoped_lock` 的以 `m` 为参数的构造函数，然后将这个变量销毁。**二是使用默认构造函数，声明了一个名字为 `m` 的 `std::scoped_lock` 对象。也就是等于**

    ```cpp hl_lines="2"
    void func() {
        std::scoped_lock m;
    }
    ```

    在这种有歧义的语法下，C++ 会遵守的规则是：**如果一条语句看上去是声明，那么它就是声明。**也就是说实际上如果不给对象起名字的话，上面的代码实际上会按照**第二种**理解去编译。在这种情况下，互斥量根本没有上锁。


### 原子量

我们在上文介绍过竞态条件的产生源于多个线程执行的相对顺序无法确定，使用互斥量可以确保在任意时候临界区最多只有一个线程在访问，进而就能够约束线程的执行顺序，以此来解决竞态条件问题。不过，能够约束线程执行顺序的还有另一种方法：**原子量（atomic variables）。**

原子量从英文名就能看出，它本质上是作为一种变量来供人们使用，但它却有一个很明显的特性：原子性。在 C++ 中，原子性的含义是：**多个并发线程访问具有原子性的对象时不会造成数据竞争，也不会导致未定义行为。**相比起互斥量，原子量使用另一种截然不同的思路去约束线程执行的顺序。

如果将上述使用互斥量的例子改为使用原子量，我们可以将线程竞争访问的 `index` 变量改为原子变量：

=== "with_mutex.cpp"

    ```cpp hl_lines="2"
    int list[100];
    std::atomic_int index = -1; // 使用原子量作为 `index`

    void producer() {
        while (true) {
            int data = read_data();
            list[1 + index.fetch_add(1)] = data; 
        }
    }

    void consumer() {
        while (true) {
            int i = index;
            if (i < 0) continue;
            while (!index.compare_exchange_weak(i, i - 1)) {
               if (i < 0) break;
            }
            if (i < 0) continue;
            do_calculate(list[i]);
        }
    }
    ```
    
上面的代码看上去比互斥量要稍微复杂一些，我们先来看看其中 `fetch_add()` 函数和 `compare_exchange_weak()` 函数的使用：

* `fetch_add()` 函数的作用是：原子地获取 `index` 的值，然后将它加一，再存入 `index` 中。这整个过程是原子的，不会有其它线程对 `index` 的访问穿插在这几步之间，导致数据竞争的出现。上面代码中的 `index.fetch_add(1)` 其实就是普通变量里 `i++` 的原子量版本。
* `compare_exchange_weak()` 函数的作用是：比较原子量当前的值和给定的值（第一个参数），如果相等则将原子量的值设为第二个参数的值并返回 `true`；如果不相等则将第一个参数的值设为原子量的值，并返回 `false`。此函数做的事情也是原子的。
  我们首先使用 `int i = index;` 将原子量当前的值读入本地变量 `i` 里，然后我们准备对它进行减一操作，然后消费新的数据。不过事情并没有这么简单。**在读取 `index` 到修改 `index` 之间，有可能另一个线程也做了同样的事情，导致数据竞争的出现。**为此，C++ STL 为原子量提供了一个功能强大的 `compare_exchange_weak()` 函数，用来给当前函数检查**是否有其他线程先于自己一步完成了对原子量的修改。**
  举个例子，假如现在 `producer()` 将 `index` 的值设为了 `0`，有两个消费者线程 A 和 B 都进行了 `int i = index`，读到了 `i` 均为 `0`。此时，A 进行 `compare_exchange_weak()` 函数，将 `index` 的值设为了 `i - 1` 也就是 `-1`，并调出了循环，成功地消费了此数据。B 在进行相同函数时，发现 `index` 当前的值已经不再等于它先前拿到的 `i` 了，于是 `i` 被设置为 `i - 1`，也就是 `-1`，并先后经过 `break` 和 `continue` 重新开始新一轮消费循环。像这样通过引入原子量，我们在多个并发线程对 `index` 的访问之间引入了对访问顺序的约束，避免了数据竞争的产生。

原子量和互斥量在对待多线程并发访问造成的冲突时，在理念上存在本质的区别：前者是先假定没有发生冲突，在发生冲突时重试，所以我们会在循环中进行 CAS；后者则是默认每一次都会冲突，所以每一次访问临界区都要加锁。有人因此就说使用原子量进行并发控制是乐观的（optimistic），它认为发生冲突较为少见，而使用互斥量则是悲观的（passive），它认为每一次访问都可能有冲突。

经验表明，在很多场景下使用原子量的性能一般会好于使用互斥量的性能，不过这个问题非常复杂，也有很多相反的情况。它们的性能和程序代码的写法、编译器、操作系统、CPU 架构等因素都有关系，这些已经超出了本文的讨论范围，如果你有兴趣可以自行查阅相关资料。

???+info
    `compare_exchange_weak()` 是 C++ 11 STL 提供的 **CAS（compare and set）**操作的实现之一，另一个实现是 `compare_exchange_strong()`，二者的区别在于：前者一般会用在 `while` 循环里，因为此函数有可能会出现“假阴性”现象，也就是第一个参数的值和原子量的值明明一样，函数却返回了 `false`（但不会修改第一个参数的值）。之所以有这种现象是为了更好的性能考虑。后者则不会有这种现象，因此不需要用在循环里，但是相对应的性能就会差一些。
    
???+info
    原子量提供的这些函数都可以接收额外的 `memory_order` 参数，它是 C++ 提供给程序员来指示编译器对代码的重排程度的标识。在上面的例子里，我们其实可以使用 `memory_order_consume` 和 `memory_order_acquire` 来进一步提升原子量访问的性能。更多有关 Memory order 的话题已经超出了本文的讨论范围，相关资料可以查阅 [cppreference.com](http://www.cplusplus.com/reference/atomic/memory_order/)。

### 条件变量

如果我们仔细思考前面两个例子，它们分别使用互斥量和原子量来防止数据竞争。但是在生产者还没有产生新数据，且 `list` 为空的时候，所有的线程都在不停地通过循环来检查有没有新的数据，这种行为被称为**忙等（busy waiting）**，也就是说它们虽然在等待新的数据，但仍然在通过循环来不断地消耗 CPU 时间。

如果我们能够通过某种机制，让消费者线程在等待的时候不再消耗 CPU 时间（例如进入休眠态），并且生产者在产生新数据时自动唤醒其中一个休眠的消费者线程，那么我们就能够进一步提升 CPU 的利用效率了。因为 CPU 再也不需要花时间到各个消费者线程，反复地执行循环来检查新数据。人们为此提出了**条件变量（condition variable）**。

从本质上来说，条件变量就是一个先进先出队列，队列中存储的是等待的线程。生产者线程可以通过 `notify_one()` 函数来取出队头的线程并唤醒它；也可以通过 `notify_all()` 函数来取出所有线程并唤醒它们。

我们使用条件变量来改善上文中基于互斥量的例子：

=== "with_cv.cpp"

    ```cpp hl_lines="4 14 23"
    int list[100];
    int index = -1;
    std::mutex m;
    std::condition_variable cv; // 声明条件变量

    void producer() {
        while (true) {
            int data = read_data();
            {   // 使用 scoped_lock 来自动管理互斥量
                std::scoped_lock lk(m);
                list[index + 1] = data; 
                index++;
            }
            cv.notify_one(); // 唤醒等待队列里的一个消费者线程
        }
    }

    void consumer() {
        while (true) {
            int data;
            {
                std::unique_lock<std::mutex> lk(m);
                cv.wait(lk, [] { return index >= 0; });
                // 从这里开始，条件变量保证：此时满足 index >= 0 且持有互斥量
                data = list[index--];
            }   // 离开此作用域时，互斥量自动被释放
            do_calculate(data);
        }
    }
    ```

注意 `consumer()` 中的 `cv.wait()` 调用。它的作用是：将当前线程加入等待队列，然后阻塞当前线程。当线程被唤醒后，使用给定的 `unique_lock` 并检测输入的函数的返回值，当返回值为 `true` 时函数返回，程序继续执行；否则，再次阻塞当前线程。因此，C++ 中的互斥量总是会搭配一个互斥量进行使用。

由于各种原因，通过 `wait()` 而被阻塞的线程有可能会在条件变量并没有被调用 `notify()` 等函数时被唤醒，这种现象被称为 spurious wakeup。此时等待线程会拿锁，然后检查输入的函数的返回值，此时返回值一般为 `false`（因为此时确实没有新数据），然后再一次进入阻塞状态。在这个过程中没有发生数据竞争。

条件变量的设计天然地防止了问题的产生。我们还可以尝试将 `producer()` 中的 `cv.notify_one()` 改为 `cv.notify_all()`，它会唤醒所有等待线程，但与此同时我们只输出了一个新数据。结果是：等待线程只会有一个拿到这个新数据，其他线程会再一次进入阻塞状态，仍然没有数据竞争的出现。
