class Socket
```cpp
class Socket:
    muduo::net socket
    
    explicit Socket(int sockfd); // 初始化socket fd，传入fd，无需socket()返回
    ~Socket(); // 关闭fd
    
    int fd() const { return sockfd_; }  // 返回fd
    bool getTcpInfo(struct tcp_info*) const;
    bool getTcpInfoString(char* buf, int len) const;
    void setTcpNoDelay(bool on); // setsockopt设置tcp信息
    void Socket::setReuseAddr(bool on); // setsockopt
    void Socket::setReusePort(bool on); // setsockopt
    void Socket::setKeepAlive(bool on); // setsockopt
        
    // 以下函数调用sockets接口 -- 没有write和read函数
    void bindAddress(const InetAddress& localaddr);  // socket::bind
    void listen();
    int accept(InetAddress* peeraddr); // 返回值为接入fd，参数为介入fd的addr信息
    void shutdownWrite(); 
    
    const int sockfd_;
    
```
