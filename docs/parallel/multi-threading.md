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