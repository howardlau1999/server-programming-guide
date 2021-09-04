# 使用 Prometheus 指标库

Prometheus 是常用的指标库之一，它只需要开发者提供一个 HTTP 接口，在这个接口使用纯文本返回一系列指标值即可，例如：

```
requests_served 12345
uptime 25.23
operations{ops="GET"} 234
```

可以看到，这就是一个简单的带标签的 Key 到 Float 的映射。我们可以自己整合 HTTP 服务器来提供这个接口，也可以使用现成的库来提供。