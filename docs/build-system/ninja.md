# Ninja

Ninja 是 make 的替代品，使用了不一样的 DSL，构建文件通常命名为 `ninja.build`。Ninja 宣称在依赖解析和增量构建时会比 make 更快，同时默认采用了并行编译，无需像 make 那样通过 `-j` 选项来开启并行。

Ninja 的构建文件语法比 makefile 更简单，功能也更弱，如果让人类来手写的话会非常长。因此，我们一般不手写 Ninja 文件，而是通过元构建系统来生成，例如 [GN](../meta-build-system/gn.md)。Ninja 在 Windows 上也是可以使用的。