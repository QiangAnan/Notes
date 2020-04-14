
## tcpServer : class



## tcpClient : class
```cpp
// 成员变量
    EventLoop* loop_;   // 构造时传入
    ConnectorPtr connector_; // class Connector, tcpClient构造时new Connector
    ConnectionCallback connectionCallback_;
    MessageCallback messageCallback_;
    WriteCompleteCallback writeCompleteCallback_;
    TcpConnectionPtr connection_ GUARDED_BY(mutex_); // TcpConnection


```
