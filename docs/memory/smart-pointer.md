# 智能指针

在现代 C++ 编程中，为了保证内存安全以及防止内存泄漏，应该尽量减少裸指针的使用，也尽量不要使用 `#!cpp new` 和 `#!cpp delete`，而应该使用**智能指针**。智能指针利用了 C++ RAII 的特性，在智能指针对象生命周期结束时，使用析构函数自动释放指针所对应的内存。最常用的智能指针有 `#!cpp std::unique_ptr`、`#!cpp std::shared_ptr` 以及 `#!cpp std::weak_ptr`。

## `#!cpp std::unique_ptr`

`std::unique_ptr` 表示一个指针最多只能被引用一次，也就是不存在共享的关系。使用时应该使用 `std::make_unique` 来生成一个指向对象的 `std::unique_ptr`。`std::unique_ptr` 不允许拷贝，也就是说传递 `std::unique_ptr` 的时候只能使用 `std::move` 来使用移动语义。

???+warning
    智能指针永远都是按值传递，不可以使用引用的方式。

## `#!cpp std::shared_ptr`

`std::shared_ptr` 表示一个指针可能被多次引用，也就是存在着共享的关系。使用时应该使用 `std::shared_ptr` 来生成一个指向对象的 `std::shared_ptr`。`std::shared_ptr` 允许拷贝，在使用时直接使用按值传递的方式来使用即可，不需要使用 `std::move`。

`std::shared_ptr` 中内置了引用计数，当发生拷贝的时候就会给引用计数加一，析构函数则会减一。当引用计数为 0 说明这个指针已经无人使用了，就会调用析构函数去释放内存。

???+warning
    再次强调，智能指针永远都是按值传递，不可以使用引用的方式。
