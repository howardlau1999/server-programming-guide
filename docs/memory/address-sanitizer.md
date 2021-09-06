# 使用 AddressSanitizer 检查内存安全问题

[AddressSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer) 是 Google 开发的内存错误检测器，可以检查缓冲区溢出以及访问野指针等内存错误。目前，较新版本的 GCC 编译器以及 clang 编译器都已经内置 AddressSanitizer 的支持。使用了 AddressSanitizer 的程序通常会比不使用的慢 2 倍左右。

要使用 AddressSanitizer，只需要使用编译器的 `-fsanitize=address` 选项即可。例如，有一个数组越界的程序：

=== "overflow.cpp"
    ```cpp
    #include <iostream>
    #include <vector>

    int main() {
        std::vector<int> v{1, 2, 3};
        std::cout << v[10000] << std::endl;
        return 0;
    }
    ```

打开编译器的 AddressSanitizer 开关编译后运行，就能看到程序在访问数组越界的时候被终止并输出错误信息了：

```bash
$ g++ -g -fsanitize=address overflow.cpp
$ ./a.out
=================================================================
==10814==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x602000009c50 at pc 0x55cd5d020679 bp 0x7ffc59648d50 sp 0x7ffc59648d40
READ of size 4 at 0x602000009c50 thread T0
    #0 0x55cd5d020678 in main /home/howard/overflow.cpp:6
    #1 0x7f92281e6cb1 in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x28cb1)
    #2 0x55cd5d02038d in _start (/home/howard/a.out+0x238d)

Address 0x602000009c50 is a wild pointer.
SUMMARY: AddressSanitizer: heap-buffer-overflow /home/howard/overflow.cpp:6 in main
# ...
```

除了不安全的内存访问之外，AddressSanitizer 也可以检测出内存泄露的问题，例如：

=== "leak.cpp"
    ```cpp
    int main() {
        int* arr = new int[100];
        return 0;
    }
    ```

同样打开 AddressSanitizer 编译运行：

```bash
$ g++ -g -fsanitize=address leak.cpp
$ ./a.out

=================================================================
==11173==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 400 byte(s) in 1 object(s) allocated from:
    #0 0x7ff377401097 in operator new[](unsigned long) ../../../../src/libsanitizer/asan/asan_new_delete.cpp:102
    #1 0x55b65daf319e in main /home/howard/leak.cpp:2
    #2 0x7ff376e41cb1 in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x28cb1)

SUMMARY: AddressSanitizer: 400 byte(s) leaked in 1 allocation(s).
```