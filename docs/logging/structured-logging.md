# 结构化日志

基于格式字符串格式化的纯文本日志在分析的时候通常还是需要使用正则表达式或者语法解析器来提取日志中的信息。那么，既然我们还是需要使用工具来分析日志，那么为什么不直接输出工具可以解析的格式呢？例如，我们可以将日志输出成 JSON 格式：

例如：

```cpp
LOG(INFO) << "Test Logging userID = " << userID << ", username = " << username;
```

可以变成：

```cpp
logger.info("Test Logging", Int("user-id", userID), String("username", username));
```

让日志库输出：

```json
{"level":"info","ts":1614743088.538128,"caller":"main.cpp:15","user-id":1,"username":"howard","func":"test","msg":"Test Logging"}
```

使用这种方式，我们不再将变量信息根据格式字符串编码成一段纯文本，而是将变量信息视为 KV 数据对，附加到信息中，这样的日志人类可读，也有现成的解析器来解析，方便处理和检索。