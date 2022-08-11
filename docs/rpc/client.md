# 编写客户端

RPC 客户端主要的功能是连接到服务端，发送请求并接收响应。客户端需要实现的逻辑几乎没有，只需要根据 IDL 定义的函数签名构造好请求就可以了。复杂度主要在于客户端需要负责超时重试，以及连接复用避免大量创建连接的开销。其中，连接复用通常是通过连接池的方法来实现的。

总体来说，客户端的伪代码如下：

```cpp

class RPCClient {
  ConnectionPool pool_;
  Response do_request(Request const& req) {
    auto conn = pool_.get();
    conn.send_request(req);
    auto resp = conn.recv_response();
    pool_.put(conn);
    return resp;
  }
public:
  RPCClient(string const &host, string const &port) : pool_(host, port) {}
  PingResponse ping(PingRequest const& req) {
    return do_request(req);
  }
};
```

连接池预先建立并维护了一些空闲连接，在发送请求的时候，直接从连接池中取出一个连接，在这个连接完成一次 RPC 之后，将连接重新放入连接池，等待下一次 RPC 调用。为了避免占用过多的服务器资源，连接池通常会设定一个空闲超时，将空闲时间太久的连接关闭。同时，连接池会设定一个最大连接数，如果连接数超过了这个数，后续的 RPC 调用则需要等待直到有可用连接为止。对于连接出错的情况，通常是直接关闭连接，并尝试重新创建一个新的连接。当然，重试也有超时和次数上限，如果超时或者次数超过了上限，则直接抛出异常。