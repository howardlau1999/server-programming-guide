# 编写服务端

RPC 服务端和普通的服务器程序一样，监听一个服务端口，接收并响应客户端的请求。常用的 RPC 协议有 JSON-RPC 和 gRPC。对于后台服务之间的通信，使用基于二进制格式的通信协议效率更高，而如果服务器需要面向浏览器提供 API 服务的话，使用 JSON 格式的通信协议会提高前端的开发效率。

通常来说，一个 RPC 服务器的伪代码如下：

```cpp
class RPCServer {
  std::vector<Middleware> middlewares_;
public:
  response ping(request &req) {
    return response(req.body);
  }
  response handle_request(request &req) {
    switch (req.method) {
    case kMethodPing:
      return ping(req);
    // ...
    }
  }
  task handle_connection(connection &conn) {
    while (conn.alive()) {
      auto req = conn.read_request();
      for (auto &m : middlewares_) {
        req = m.handle_request(conn, req);
      }
      auto resp = handle_request(req);
      conn.write_response(resp);
    }
  }
  void serve(int port) {
    auto listener = listen_on(port);
    while (!stopped) {
      auto conn = listener.accept();
      handle_connection(conn).detach();
    }
  }
};
```

其中，服务端先启动一个监听端口，然后不断接受连接。我们将每一个接受到的连接放到后台去处理，在连接还没断开的时候，接受并响应请求。而响应请求的逻辑也很直接，我们根据请求中的头部信息，获取请求对应的方法，然后调用对应的处理函数获得响应后，将响应发送回去。对于一些通用的逻辑，例如参数校验、鉴权、限流等可以抽象成为**中间件**，对请求进行一些预处理之后再调用真正的处理函数。响应的回复也可以用类似的思想，对响应进行一些统一的处理。

由于编写框架代码比较繁琐，我们可以使用代码生成的方式让工具来帮助我们生成这些模板代码。例如 Protobuf 就支持根据 IDL 定义生成 RPC 服务端和客户端的框架代码，程序员只需要实现预定义好的接口，就完成了 RPC 程序的编写。
