## 对特定线程打印断点

在 gdb 中，默认打断点会应用到全部线程上，这时候如果多个线程同时执行了同样的代码，那么断点就会在不同线程来回切换，不方便跟踪。gdb 可以使用 `break xxx thread yyy` 的语法，只对某个线程打断点[^1]。

## 参考资料

[^1]: [https://sourceware.org/gdb/onlinedocs/gdb/Thread_002dSpecific-Breakpoints.html](https://sourceware.org/gdb/onlinedocs/gdb/Thread_002dSpecific-Breakpoints.html)