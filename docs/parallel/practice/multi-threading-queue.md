# 概览

支持多线程共享的队列是一种相当常见且重要的数据结构。正如我们在[多线程编程](../multi-threading.md)中看到的那样，需要多线程的场景很多都会表现为一种生产者和消费者之间的协作关系。为了让生产者得以将数据传递给消费者，我们往往就需要一个先进先出（FIFO）的数据结构：队列。

在传统的单线程程序里，实现一个队列相当简单。不过在多线程的环境下，除了无死锁（deadlock-free）等基本条件之外，我们的队列还必须满足以下额外的条件：

* 线程安全（thread-safe）

  多个线程在并发访问数据结构时，不会出现竞态条件，也不会造成数据结构内部状态的损坏。

* 异常安全（exception-safe）

  当一个线程调用数据结构提供的函数中途出现异常时，数据结构内部状态不会损坏，其他线程依然可以正常访问并得到正确的结果。并且也不会出现内存泄露。
  
  虽然异常安全在单线程场景下也很重要，但它值得在多线程场景中被再次强调。因为多线程下因异常安全造成的 bug 往往更加难以复现、排查，有时甚至会造成整个软件的崩溃。

在 C++ 中实现这样的一个队列，至少存在两种方法：基于互斥量的方法和基于原子量的方法。为了简单起见，我们在本节中只介绍基于互斥量的方法。

而根据生产者和消费者的数量不同，可以分为四种队列：

- 单生产者单消费者（SPSC）队列
- 单生产者多消费者（SPMC）队列
- 多生产者单消费者（MPSC）队列
- 多生产者多消费者（MPMC）队列

多线程中使用得比较多的是 MPSC 队列以及 MPMC 队列。区分这么多种队列是因为在消费者或者生产者只有一个的时候，可以编写出并发度和性能更高的队列。本节介绍的是 MPMC 队列。

## 简单的粗粒度锁实现

C++ 的 STL 中已经提供了一个队列：`std::queue`，虽然它提供了一定的异常安全保证，但它不是线程安全的。要使用互斥量来让它变得线程安全，一个简单的方式是使用互斥量保护所有的函数：

```cpp
template<typename T>
class threadsafe_queue {
    std::queue<T> q;
    mutable std::mutex m;

public:
    threadsafe_queue() {}

    bool empty() const {
        std::scoped_lock lock(m);
        return q.empty();
    }

    void push(T element) {
        std::scoped_lock lock(m);
        q.push(std::move(element));
    }

    bool pop(T& element) {
        std::scoped_lock lock(m);
        if (!q.empty()) {
            element = q.front();
            q.pop();
            return true;
        }
        return false;
    }
};
```

在上面的例子中，我们在各个函数中访问 `std::queue` 对象之前都使用互斥量来确保竞态条件不会发生，实现了一个线程安全队列。

???+note
    对于不是线程安全的数据结构，我们都可以简单地使用互斥量将并行访问转变为串行访问，从而实现线程安全。虽然这种方式并发度不高，但是容易调试，在开发时间不够或者开发人员还不熟练的情况下，也是一种可行的方法。

???note "异常安全的分析"
    虽然我们的实现基于具有一定异常安全性的 `std::queue`，但是在包装的过程中有可能会引入一些新的破坏了异常安全性的地方。我们不妨对三个函数稍作检查。

    * `empty()` 函数是异常安全的。`std::scoped_lock` 和 `std::mutex` 的 `lock()` 函数都提供了基本的异常安全保证，即在出现异常时互斥量仍处于有效状态，其他线程可以正常使用。

    * `push()` 函数中调用的 `q.push()` 在底层会调用内部容器（通常是 `std::deque` 或 `std::list`）的 `push_back()` 函数。

    这里，STL 保证 `push_back()` 在因为内存分配出错或者元素拷贝或移动出错时，会表现得就像函数没有被调用过一样。在出现异常后，C++ 会自动进行栈展开（stack unwinding），`lock` 的析构函数被调用，我们的锁因此被解开。
    
    因此，我们的 `push()` 函数是异常安全的。

    * `pop()` 函数中，注意到 `pop` 函数中的 `element = q.front()` 语句，在这里我们对 `T` 类型的实例进行了拷贝复制操作。
    
    这里的 `T` 是由用户指定的类型，**并没有保证拷贝复制操作的过程不会出现异常**。当赋值操作不幸地出错时，C++ 自动进行栈展开，`lock` 析构函数被调用，锁被解开。因此，我们的 `pop()` 函数异常安全。

上述代码中实现的 `threadsafe_queue` 是一个基本实现了线程安全和异常安全的队列数据结构。但是它的锁粒度过粗，临界区比较大，并发度比较低。在特定场景下我们可以允许更多的并发性，而不是限制同一时刻只有一个线程访问数据结构。要想提高并发度，就要想办法将互斥锁拆分为更细粒度的锁，也就是尽量缩小临界区。

假设我们的 `std::queue` 底层使用了 `std::list` 作为内部容器，在队列内部拥有一个具有*一定数量*节点的链表时，一对并发的 `push()` 和 `pop()` 调用并不会让队列操作同一个链表节点，因此不会出现竞态条件。**如果能让队列尽可能细致地知道何时应该使用互斥量、何时不应该使用，那么就能进一步提升并发访问性能了。**同时，每次持有锁的时间越短，也能进一步提升性能。

为了编写使用细粒度锁的队列，我们需要放弃简单包装 `std::queue` 的方法，自行实现一个链表数据结构，在它的基础上使用细粒度锁来实现尽可能高的并发访问性能。

## 基于链表的细粒度锁实现

为了方便理解，我们接下来会一步一步地构建出一个满足上述要求的队列。我们先来考虑一个非线程安全的实现：

```cpp
template<typename T>
class queue {
    struct node {
        T element;
        std::unique_ptr<node> next;

        node(T element) : element(std::move(element)) {}
    };

    std::unique_ptr<node> head;
    node* tail;

public:
    queue() : tail(nullptr) {}
    // 为了简便，我们暂时不考虑拷贝构造和赋值
    queue(const queue& other) = delete;
    queue& operator=(const queue& other) = delete;

    void push(T element) {
        auto ptr = std::make_unique(std::move(element));
        auto new_tail = ptr.get();
        if (tail == nullptr)
            head = std::move(ptr);
        else
            tail->next = std::move(ptr);
        tail = new_tail;
    }

    std::shared_ptr<T> pop() {
        if (!head)
            return std::shared_ptr<T>();
        auto res = std::make_shared(std::move(head->data));
        auto old_head = std::move(head);
        head = std::move(old_head->next);
        if (!head)
            tail = nullptr;
        return res;
    }
};
```

在上面的例子里，我们使用 `std::unique_ptr` 来管理链表的节点存储，使用裸指针 `tail` 来指向尾节点。使用[智能指针](../../memory/smart-pointer.md)可以有效防止内存泄露的问题。这个例子并不是线程安全的，因为在队列里只有一个节点的时候，并发调用的 `push()` 和 `pop()` 有可能同时访问同一个节点的 `next` 指针，从而出现竞态条件。你可能会认为如果在两个函数里判断一下这种情况并加锁就能解决问题，就像下面这样：

```cpp
auto has_single_item = !empty() && head->get() == tail;
auto lock = has_single_item ? std::scoped_lock(m) : std::scoped_lock();
```

不幸的是，这种解决方法是错误的。我们随便就能举一个能够绕过这种保护措施的例子：存在三个线程，其中两个进行 `pop()`，另一个进行 `push()`。假如队列里有两个元素，当三个线程都执行完 `has_single_item` 的运算时得到了 `false`，于是三个线程都不会去获得锁从而继续访问，竞态条件因此出现。

除了对同一个节点进行 `push()` 和 `pop()` 可能导致的竞态条件，我们还要考虑并发访问同一个函数（`push()` 或者 `pop()`）的情况。这么考虑下来，我们最后写出的代码可能还不如一开始就对 `std::queue` 使用互斥量包装。

要想真正地解决上述问题，我们需要转变思维。既然在节点数量为一的时候，并发的 `push()` 和 `pop()` 调用会造成竞态条件，那么我们可以引入一个空节点来让链表节点数量总是大于等于一。**这样在队列里只有一个数据时，链表会含有两个节点，此时并发的 `push()` 和 `pop()` 调用就不会造成竞态条件了。**请看下面的代码：

```cpp hl_lines="26"
template<typename T>
class queue {
    struct node {
        std::shared_ptr<T> element;
        std::unique_ptr<node> next;
    };

    std::unique_ptr<node> head;
    node* tail;

public:
    queue() : head(std::make_unique<T>()), tail(head.get()) {}
    // 为了简便，我们暂时不考虑拷贝构造和赋值
    queue(const queue& other) = delete;
    queue& operator=(const queue& other) = delete;

    void push(T element) {
        auto element_ptr = std::make_shared(std::move(element));
        auto node_ptr = std::make_unique<T>();
        auto new_tail = node_ptr.get();
        // 将当前尾节点的值设为 `element_ptr`，而新建的空节点 `node_ptr` 作为新的尾节点
        tail->data = element_ptr;
        tail->next = std::move(node_ptr);
        tail = new_tail;
    }

    std::shared_ptr<T> pop() {
        if (head.get() == tail)
            return std::shared_ptr<T>();
        auto res = head->data;
        auto old_head = std::move(head);
        head = std::move(old_head->next);
        return res;
    }
};
```

为了引入空节点的设定，我们将 `struct node` 里的 `element` 改为了 `std::shared_ptr`。这样对于空节点来说，智能指针内容就为空。然后，`empty()` 的判断依据改为了 `head.get() == tail`，因为在链表只有一个节点（也就是空节点）时我们认为队列为空。`pop()` 函数的改动不大，因为现在链表在 `pop()` 之后不可能为空，所以不需要额外的判断了。

最值得讨论的是 `push()` 的修改。我们在这里的策略是：**将用户输入的值存到当前作为尾节点的空节点，然后再新建一个空节点附加到链表尾部。**这样一来，我们发现即使队列里只有一个数据，链表里仍会有两个节点，此时并发的 `pop()` 和 `push()` 调用再也不会访问同一个节点了。

接下来，我们在这种设计的基础上放置互斥量。和简单的实现不同，我们追求尽可能大的并发访问性。经过分析，我们可以发现这些可能造成竞态条件的情况：

1. 多个线程并发调用 `push()`
2. 多个线程并发调用 `pop()`
3. 多个线程并发调用 `push()` 和 `pop()`

这几种情况的竞争焦点都是 `head` 和 `tail`，我们可以分别给它们引入一个互斥量进行保护。最后的代码如下所示：

```cpp
template<typename T>
class queue {
    struct node {
        std::shared_ptr<T> element;
        std::unique_ptr<node> next;
    };

    std::unique_ptr<node> head;
    node* tail;
    std::mutex head_mutex;
    std::mutex tail_mutex;

    node* get_tail() {
        std::scoped_lock lock(tail_mutex);
        return tail;
    }

    std::unique_ptr<node> pop_head() {
        // 给 `head` 加锁，防止并发的 `pop()` 导致竞态条件
        std::scoped_lock lock(head_mutex);
        // `get_tail()` 会给 `tail` 加锁，从而避免了并发的 `pop()` 和 `push()` 可能导致的竞态条件
        if (head.get() == get_tail())
            return std::unique_ptr<node>();
        auto old_head = std::move(head);
        head = std::move(old_head->next);
        return old_head;
    }

public:
    queue() : head(std::make_unique<T>()), tail(head.get()) {}
    // 为了简便，我们暂时不考虑拷贝构造和赋值
    queue(const queue& other) = delete;
    queue& operator=(const queue& other) = delete;

    void push(T element) {
        auto element_ptr = std::make_shared(std::move(element));
        auto node_ptr = std::make_unique<T>();
        auto new_tail = node_ptr.get();
        // 仅在需要访问 tail 时才加锁，这里的锁可以避免并发的 `push()` 导致的竞态条件
        std::scoped_lock lock(tail_mutex);
        tail->data = element_ptr;
        tail->next = std::move(node_ptr);
        tail = new_tail;
    }

    std::shared_ptr<T> pop() {
        auto old_head = pop_head();
        if (old_head)
            return old_head->data;
        else
            return std::shared_ptr<T>();
    }
};
```

从上面的代码来看，我们通过引入两个互斥量，成功地解决了可能出现的竞态条件。虽然我们最终还是引入了互斥量，但是对比上面的例子我们可以看到：**`push()` 和 `pop()` 只在恰当的时候才给互斥量上锁，临界区变小了，并发性能得到了提升。**上面的实现依然是异常安全的，至于具体的分析则留给读者。

## 小结

在本节中我们了解到了怎样编写以及分析一个线程安全的同步队列。同步队列是线程间通信的基本数据结构，你能在几乎所有多线程程序中见到同步队列的实现。也因为同步队列是最基础的数据结构之一，大量开发者投入了大量精力来优化同步队列的性能，本节介绍的是使用互斥量实现的同步队列，还有许多只使用原子操作实现的无锁队列，在对性能极度敏感的场景下可以使用。

???+info
    你可能会了解到其他语言中有着其他的同步方法，例如 Go 语言中的 Channel、Erlang 语言中的 Mailbox，但其背后的实现原理仍然是同步队列，因此，在编写多线程以及多进程程序的时候，必须熟练掌握队列这一基本而重要的数据结构。

## 参考实现

以下是一些健壮的高性能的多线程队列实现参考：

- [Boost 的无锁队列](https://www.boost.org/doc/libs/1_77_0/doc/html/boost/lockfree/queue.html)
- [Intel Thread Building Blocks](https://spec.oneapi.io/versions/latest/elements/oneTBB/source/containers/concurrent_queue_cls.html)
- [moodycamel](https://github.com/cameron314/concurrentqueue)