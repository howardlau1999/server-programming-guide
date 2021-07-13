# 控制反转

通常我们的服务器中，需要调用许多外部服务。为了实现更灵活的配置以及更好的解耦性，在使用外部服务时，我们会使用控制反转的思想来编写代码。控制反转的一个重要思想就是，使得程序尽可能依赖**接口**，而不是具体**实现**。

那么，控制反转是什么意思呢？假如我们的程序 A 需要使用 B 服务，那么在没有控制反转的情况下，我们会在程序里写出以下代码：

```cpp
// B.h 中提供了服务 B 的实现
#include "B.h"

class MyServer {
    std::unique_ptr<B> serviceB;
public:
    MyServer() {
        serviceB = std::make_unique<B>();
    }

    void Work() {
        serviceB->xxx();
    }
};
```

也就是说，我们需要用到什么服务，就自己初始化。这时候我们的程序 A 需要依赖 B。很显然，这样相当于把我们的服务和其他程序耦合了。另外，测试也没有那么方便，测试的时候需要 B 服务正常运行才可以进行测试。

而使用控制反转，我们将借助接口，从而实现使用者与服务提供者的解耦。我们首先需要在程序 A 中声明我们希望服务提供什么样的接口，然后，在构造程序 A 的时候，通过构造函数，将实现了接口的对象传递进来，而不是由 A 自己构造。

=== "BInterface.h"

    ```cpp
    class BInterface {
    public:
        virutal void xxx() = 0;
    };
    ```
=== "B.cpp"

    ```cpp
    #include "BInterface.h"

    class ServiceB : public BInterface {
    public:
        virtual void xxx() {
            // ...
        }
    };
    ```

=== "Server.cpp"

    ```cpp
    #include "BInterface.h"

    class MyServer {
        std::shared_ptr<BInterface> serviceB;
    public:
        MyServer(std::shared_ptr<BInterface> serviceBImpl) {
            serviceB = serviceBImpl;
        }

        void Work() {
            serviceB->xxx();
        }
    };
    ```

此时，依赖的方向变为了服务 B 依赖 A 提供的接口定义，这样就实现了控制的反转。另外，由于服务对象不再由 A 自己构造，我们可以很方便的在测试或者上线的时候，给 A 程序提供不同的服务实现。比如，我们可以通过配置文件来实现灵活切换不同的服务实现。

```cpp
#include "BInterface.h"
#include "ServiceBv1.h"
#include "ServiceBv2.h"

std::shared_ptr<BInterface> serviceB;

if (config->useServiceBv1) {
    serviceB = std::make_shared<ServiceBv1>();
} else {
    serviceB = std::make_shared<ServiceBv2>();
}

MyServer server(serviceB);
```

而测试时，我们可以提供一个测试用的服务。

```cpp
#include "BInterface.h"

class MockServiceB : public BInterface {
public:
    virtual void xxx() {
        // ...一个简单的用于测试的实现
    }
};

void TestMyServer() {
    MyServer server(std::make_shared<MockServiceB>());
    // ...测试代码
}
```

在编程时，尽量养成先定义接口，然后再具体实现的习惯。使用接口能帮助我们解耦程序之间的依赖。