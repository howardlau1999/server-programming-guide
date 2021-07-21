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

这个概念比较抽象，可以举几个例子来方便理解。

## 例子：使用 KV 存储的用户服务

假如我们现在编写了一个用户服务，可以使用一个 ID 来查询或者修改用户的具体信息，为了重启之后不丢信息，我们可能会使用一个可持久化的数据库来存储数据。

而我们根据不同的需要，可能会使用可持久化的 KV 存储（可以理解成数据保存在硬盘上的 `#!cpp std::map`），这个 KV 存储可能是保存在本地文件的（例如 [RocksDB](https://github.com/facebook/rocksdb)），也可能是另外有服务进程来提供服务（例如我们用 [Redis](https://github.com/redis/redis)），而在测试的时候，我们不想这么麻烦，可能就直接使用内存里的写死的测试数据来进行测试。而且，未来可能出现了更新更强的 KV 存储。那么，我们的服务就不应该写死具体的类，而是使用**接口**的方式去解耦依赖。

那么，此时应该去抽象出我们服务所需要的最少的功能，作为我们的依赖接口。例如，我们这个例子中，我们的服务实际上依赖的 KV 接口是 `Get` 和 `Put`，那么我们就可以定义一个抽象的 KV 接口：

=== "kv.h"

    ```cpp
    class KV {
    public:
        virtual std::string Get(std::string key) = 0;
        virtual void Put(std::string key, std::string value) = 0;
        virtual ~KV() = 0;
    };
    ```

而我们的用户服务，则只需要依赖这个接口定义即可。

=== "user_service.cpp"

    ```cpp
    #include "user_service.h"
    #include "kv.h"
    
    UserService::UserService(std::shared_ptr<KV> kv) {
        kv_ = kv;
    }

    std::string UserService::SerializeUser(User const& user) {
        // ...
    }

    User UserService::DeserializeUser(std::string serialized_user) {
        // ...
    }

    User UserService::GetUser(std::string id) {
        std::string serialized_user = kv_->Get(id);
        return DeserializeUser(serialized_user);
    }
    
    void UserService::UpdateUser(std::string id, User const& user) {
        std::string serialized_user = SerializeUser(user);
        kv_->Put(id, serialized_user);
    }
    ```

=== "user_service.h"

    ```cpp
    #include "kv.h"

    class User {
        std::string username_;
    public:
        User(std::string const& username) : username_(username) {}
        std::string GetUsername() { return username_; }
    };

    class UserService {
        std::shared_ptr<KV> kv_;
    public:
        UserService(std::shared_ptr<KV> kv);
        User GetUser(std::string id);
        void UpdateUser(std::string id, User const& user);
        
        static std::string SerializeUser(User const& user);
        static User DeserializeUser(std::string serialized_user);
    };
    ```

那么，我们的用户服务就不依赖于特定的 KV 存储实现了。

例如，我们可以实现 RedisKV、RocksDBKV，然后在主程序中按需使用。

=== "redis_kv.h"
    
    ```cpp
    #include "kv.h"
    #include "redis.h"

    class RedisKV : public KV {
        std::shared_ptr<RedisClient> client_;
    public:
        RedisKV(std::shared_ptr<RedisClient> client);
        ~RedisKV();      
    }
    ```

=== "redis_kv.cpp"

    ```cpp
    #include "redis_kv.h"
    #include "redis.h"

    RedisKV::RedisKV(std::shared_ptr<RedisClient> client) {
        client_ = client;
    }

    RedisKV::~RedisKV() {
        client_->Close();
    }

    std::string RedisKV::Get(std::string key) {
        return client_->Get(key);
    }
    
    void RedisKV::Put(std::string key, std::string value) {
        client_->Put(key, value);
    }
    ```

=== "rocksdb_kv.h"
    
    ```cpp
    #include "kv.h"
    #include "rocksdb.h"

    class RocksDBKV : public KV {
        std::shared_ptr<RocksDBClient> client_;
    public:
        RocksDBKV(std::shared_ptr<RocksDBClient> client);        
        ~RocksDBKV();
    }
    ```

=== "rocksdb_kv.cpp"

    ```cpp
    #include "rocksdb_kv.h"
    #include "rocksdb.h"

    RocksDBKV::RocksDBKV(std::shared_ptr<RocksDBClient> client) {
        client_ = client;
    }

    RocksDBKV::~RocksDBKV {
        client_->Close();
    }

    std::string RocksDBKV::Get(std::string key) {
        return client_->Get(key);
    }
    
    void RocksDBKV::Put(std::string key, std::string value) {
        client_->Put(key, value);
    }
    ```

在主程序中，我们可以像这样使用：

=== "user_service_main.cpp"

    ```cpp
    #include "user_service.h"
    #include "rocksdb_kv.h"
    #include "redis_kv.h"
    #include "rocksdb.h"
    #include "redis.h"

    int main() {
        // ... 读取配置
        std::shared_ptr<KV> kv;
        if (config->use_rocksdb) {
            kv = std::make_shared<RocksDBKV>(NewRocksDBClient(config->rocksdb_path));
        } else if (config->use_redis) {
            kv = std::make_shared<RedisKV>(NewRedisClient(config->redis_addr));
        }
        UserService user_service(kv);

        server->OnHttpGet("/user/{id}", [&user_service](HttpRequest const& req, HttpReponse& resp) {
            resp->data = "Your username is: " + user_service.GetUser(req->GetParam("id")).GetUsername();
        })->OnHttpPost("/user/{id}", [&user_service](HttpRequest const& req, HttpReponse& resp) {
            std::string const& id = req->GetParam("id");
            std::string username = req->GetBody();
            user_service.UpdateUser(id, User(username));
        });
        
        return server->Serve();
    }
    ```

如果我们想给我们的服务编写测试，那么我们只需要编写一个测试用的 KV。

=== "test_user_service.cpp"

    ```cpp
    #include "kv.h"
    #include "user_service.h"

    class MockUserKV : public KV {
        std::map<string, string> data_;
    public:
        MockUserKV() {
            data_["1"] = UserService::SerializeUser(User("a"));
            // ...
        }

        std::string Get(std::string key) {
            return data_[key];
        }

        void Put(std::string key, std::string value) {
            data_[key] = value;
        }
    };

    TEST(UserServiceTest, GetUser) {
        UserService user_service(std::make_shared<MockUserKV>());
        EXPECT_EQ(user_service->GetUser("1").GetUsername(), "a");
    }
    ```

## 例子：使用关系型数据库的用户服务

假如我们还想我们的用户服务可以使用关系型数据库，那么单纯的 KV 抽象就不太够了，我们需要使用万能办法：增加一层间接性。

假如我们的 SQL 数据库提供了这样一个接口。篇幅限制，不再写不同数据库的示例代码了，可以按照 KV 的代码如法炮制。

=== "sqldb.h"

    ```cpp
    class SQLDB {
    public:
        virtual QueryResult Query(std::string sql) = 0;
        virtaul ~QueryResult() = 0;
    };
    ```

因为接口不一样了，我们现在有两种选择，一种是用 KV 接口将 SQL 数据库再包装一次，也就是**适配器模式**（Adaptor）。

=== "sql_kv_adaptor.h"

    ```cpp
    #include "kv.h"
    #include "sqldb.h"

    class SQLKVAdaptor : public KV {
        std::shared_ptr<SQLDB> sqldb_;
    public:
        SQLKVAdaptor(std::shared_ptr<SQLDB> sqldb);

        std::string Get(std::string key) {
            // SELECT * FROM ... WHERE key = ...
        }

        void Put(std::string key, std::string value) {
            // UPDATE ... SET ... WHERE key = ...
        }
    };
    ```

另一种办法是，我们可以抽象出一个 `UserRepository` 接口，来合并不太一样的存储服务。

=== "user_repository.h"

    ```cpp
    class UserRepository {
    public:
        virtual User GetUser(std::string id);
        virtual void UpdateUser(std::string id, User const& user);
        virtual ~UserRepository();
    }; 
    ```

这看上去和 `UserService` 的接口似乎一模一样，这是因为我们的例子比较简单。如果像 `UpdateUser` 这种函数，可能需要在修改数据前验证数据合法性以及唯一性，就不适合放在 `Repository` 层去做了。每一层都应该只有单一的职责，关注程序的不同功能方面。

而想统一接口，就需要修改我们的 `UserService` 去使用 `UserRepository` 接口，具体实现也不再写出来，可以想象为直接调用 `UserRepository` 的对应接口：

=== "user_service.h"

    ```cpp
    #include "user_repository.h"

    class UserService {
        std::shared_ptr<UserRepository> repository_;
    public:
        UserService(std::shared_ptr<UserRepository> repository);
    // ...
    };
    ```

这样，我们只需要编写不同的 `UserRepository` 实现即可：

=== "sql_user_repository.h"

    ```cpp
    #include "user_repository.h"
    #include "sqldb.h"

    class SQLUserRepository : public UserRepository {
        std::shared_ptr<SQLDB> sqldb_;
    public:
        SQLUserRepository(std::shared_ptr<SQLDB> sqldb);
        // ...
    };
    ```

=== "kv_user_repository.h"

    ```cpp
    #include "user_repository.h"
    #include "kv.h"

    class KVUserRepository : public UserRepository {
        std::shared_ptr<KV> kv_;
    public:
        SQLUserRepository(std::shared_ptr<KV> kv);
        // ...
    };
    ```

## 总结与思考

控制反转是构建大型项目时保持结构整洁的重要思想。这个思想指导我们应该**面向接口编程**，实现的细节应当依赖于接口。

但是，应该如何设计以及使用接口则没有统一的标准。过度地抽象会导致代码的嵌套层级过深，给运行性能和阅读调试代码都带来一定影响。例如，我们上面的 `UserService`，需不需要为它再单独添加一个接口类呢？这是需要权衡的。假如说一个类只有一个实现，那么为它写一个接口似乎有点画蛇添足了。

另外，一个抽象类应当包含哪些接口呢？比如说上面的 `KV`，我们需不需要提供一个可以直接帮我们处理序列化的函数呢？

软件工程中有 [SOLID 原则](https://zh.wikipedia.org/wiki/SOLID_(%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E8%AE%BE%E8%AE%A1))来指导设计接口，但这些抽象的原则是需要大量的实践与思考才能真正内化到自己的编程思维中。在实践中，并不是抽象越多越好，过度设计反而会同样造成程序阅读以及修改困难的问题。类的接口有时也不一定越少越好，太少可能会导致使用接口的人不得不编写大量重复的代码来组合接口。另外，什么时候应该使用面向对象，什么时候应该使用函数式，什么时候应该使用过程式，同样是值得权衡的问题。

我们还可以阅读优秀的开源项目来学习优秀的工程是如何组织代码以及设计接口的。不同类型的程序也往往有着不同的设计思想，适用于服务器编程的经验可能未必适用于游戏或者客户端 GUI 编程。有时候为了追求性能，我们可能也会使用“反范式”，将程序耦合在一起。不过，**抽象**仍然是计算机领域一个极为基础和重要的思想。