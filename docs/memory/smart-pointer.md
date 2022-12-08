# 智能指针

众所周知，指针是 C++ 编程中的一大难关。使用指针的时候，我们需要防止访问已经释放的内存，同时也需要避免释放同一个指针两次。在程序规模还比较小的时候，可能这不是什么大问题，但一旦函数的调用关系复杂起来，遵守上面的规则就没有这么容易了。

在现代 C++ 编程中，为了保证内存安全以及防止内存泄漏，应该使用**智能指针**，并尽量减少裸指针的使用，也尽量不要使用 `#!cpp new` 和 `#!cpp delete`。智能指针保存了内存地址的使用信息，并利用了 C++ RAII 的特性，在智能指针对象生命周期结束时，使用析构函数自动释放指针所对应的内存。最常用的智能指针有 `#!cpp std::unique_ptr`、`#!cpp std::shared_ptr` 以及 `#!cpp std::weak_ptr`。

???+warning "使用智能指针的开销"
    智能指针相比于裸指针，会带来一定程度上的额外开销。不过，只有当你编写的应用对性能**非常敏感**，才使用裸指针或引用。

## 所有权

在介绍智能指针之前，先介绍一个重要的概念：**所有权**（Ownership）。所有权就是指一个指针应该由谁“拥有”，这个所有者就需要负责释放内存。

所有权可以是唯一的，也就是同时间一个指针只能被一个对象拥有，如果需要传递指针，则需要**转移**所有权。类似于一个自习室，一次只能有一个人在房间里，这个人要么和另外一个人交换位置，要么自己离开房间并关灯。

所有权也可以是共享的，可以同时被多个对象拥有，这时候就和裸指针差不多。那么，共享所有权的指针，要怎么才能知道什么时候可以安全的使用和释放呢？最常用的方式就是**引用计数**（Reference Counting），有对象要使用这个指针了，就给引用计数加一，使用完毕了，就给引用计数减一。等到引用计数变为 0 了，就说明没有人使用这个指针了，可以安全地释放了。类似于一个很大的自习室，可以有很多人在里面，但是走的时候怎么知道房间里没有人了呢？可以在门口放一个罐子，有人进来就放一个硬币进去，有人走了就拿回一个硬币。最后拿空罐子的人就知道没有人在自习室里了，可以关灯了。当然，如果有人忘记放硬币了，那就有可能被忘在房间里了，如果有人忘记拿硬币了，房间就永远关不了灯了。

## `#!cpp std::unique_ptr`

`std::unique_ptr` 表示一个指针最多只能被引用一次，也就是不存在共享的关系。使用时应该使用 `std::make_unique` 来生成一个指向对象的 `std::unique_ptr`。`std::unique_ptr` 不允许拷贝，也就是说传递 `std::unique_ptr` 的时候只能使用 `std::move` 来使用移动语义。

## `#!cpp std::shared_ptr`

`std::shared_ptr` 表示一个指针可能被多次引用，也就是存在着共享的关系。使用时应该使用 `std::shared_ptr` 来生成一个指向对象的 `std::shared_ptr`。`std::shared_ptr` 允许拷贝，在使用时可以直接使用按值传递的方式来使用。

`std::shared_ptr` 中内置了引用计数，当发生拷贝的时候就会给引用计数加一，析构函数则会减一。当引用计数为 0 说明这个指针已经无人使用了，就会调用对象的析构函数去释放内存。

## `#!cpp std::weak_ptr`

`std::weak_ptr` 是一个具有迷惑性的名字，它实际上不是一个指针，不能像普通指针一样使用 `*` 和 `->` 来访问对象。它更像是一个兑换券，用来兑换 `std::shared_ptr`。`std::weak_ptr` 必须指向一个 `std::shared_ptr`，但是它不算在 `std::shared_ptr` 的引用计数里，也就是说哪怕还有 `std::weak_ptr` 存在，它指向的内存也有可能已经被释放了。它的使用场景比较有限，一种是打破循环引用，一种就是下面的 `this` 问题。

## `#!cpp std::enable_shared_from_this`

有的时候，我们需要在对象内部生成一个智能指针返回，例如：

```cpp
class A {
public:
    std::shared_ptr<A> get_shared_ptr() {
        return std::shared_ptr<A>(this);
    }
};

int main() {
    std::shared_ptr<A> a = std::make_shared<A>();
    {
       std::shared_ptr<A> ptr = a->get_shared_ptr();
    }
}
```

但是这样会有双重释放的问题：这个指针的引用计数被初始化为 1，使用完毕后就会释放，而这个对象自己也会调用一次析构函数。为了解决这个问题，可以使用 `std::enable_shared_from_this`：

```cpp
class A : public std::enable_shared_from_this<A> {
public:
    std::shared_ptr<A> get_shared_ptr() {
        return shared_from_this();
    }
};
```

继承了这个类之后，对象就可以使用 `shared_from_this` 方法得到指向自己的 `std::shared_ptr` 而不会有双重释放的问题了。`std::enable_shared_from_this` 类里包含一个 `std::weak_ptr` 对象，而在 `std::shared_ptr` 构造函数里，会检查这个对象是不是 `std::enable_shared_from_this` 的派生类，如果是的话，就会将构造的 `std::shared_ptr` 赋值给 `std::weak_ptr` 对象。调用 `shared_from_this` 就会调用到这个 `std::weak_ptr` 对象，从而避免了双重释放的问题。

从原理也看得出来，`enable_shared_from_this` 需要这个对象已经被一个 `std::shared_ptr` 拥有，所以要使用这个函数，必须使用 `std::make_shared` 来构造对象，否则的话，`shared_from_this` 的行为是未定义的。

## 使用智能指针的建议

正如 `new` 要和 `delete` 搭配，在使用智能指针后，我们不再需要调用 `delete`，所以，为了配平，我们也不应该调用 `new`，而是直接使用 `std::make_shared` 和 `std::make_unique`。

把智能指针当成和普通指针一样，直接传值即可，而不要传递引用。

为了避免函数和特定的智能指针类型绑定，在编写函数的时候，建议是使用对象的引用，例如 `#!cpp void func(A& a)`，这样我们就可以这样调用函数：

```cpp
std::shared_ptr<A> shared_a = std::make_shared<A>();
std::unique_ptr<A> unique_a = std::make_unique<A>();
func(*shared_a);
func(*unique_a);
```

## `#!cpp std::optional`

有些时候，我们需要使用空指针来表示“没有数据返回”等情况，例如可能我们需要查找一个数，找不到就返回空指针： `#!cpp int* find(int target)` 这样的话，使用指针的人很可能意识不到这个指针可能为空。为了进一步增强安全性，我们可以使用 `#!cpp std::optional` 这个类。

例如，`#!cpp std::optional<int>` 表示一个可能为空的整数，这样我们就不需要使用指针来表达“可能为空”的情况，从根本上避免了访问空指针的内存安全问题。
