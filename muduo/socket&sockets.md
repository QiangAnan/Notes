- class Socket
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
- moduo::net::sockets  域名中的函数接口，非class
```cpp
int createNonblockingOrDie(sa_family_t family); // 创建socket fd

// server
void bindOrDie(int sockfd, const struct sockaddr* addr);
void listenOrDie(int sockfd);
int  accept(int sockfd, struct sockaddr_in6* addr);

// client
int  connect(int sockfd, const struct sockaddr* addr);

// 收发 read write
ssize_t read(int sockfd, void *buf, size_t count);
ssize_t readv(int sockfd, const struct iovec *iov, int iovcnt); // 按照结构体iovec个数读
ssize_t write(int sockfd, const void *buf, size_t count);

// 关闭 close shutdown
void close(int sockfd);
void shutdownWrite(int sockfd);

// ip&port主机序和网络序转换
void toIpPort(char* buf, size_t size, const struct sockaddr* addr);
void toIp(char* buf, size_t size, const struct sockaddr* addr);
void fromIpPort(const char* ip, uint16_t port, struct sockaddr_in* addr);
void fromIpPort(const char* ip, uint16_t port, struct sockaddr_in6* addr);

// get本地和对端socket信息
int getSocketError(int sockfd);
const struct sockaddr* sockaddr_cast(const struct sockaddr_in* addr);
const struct sockaddr* sockaddr_cast(const struct sockaddr_in6* addr);
struct sockaddr* sockaddr_cast(struct sockaddr_in6* addr);
const struct sockaddr_in* sockaddr_in_cast(const struct sockaddr* addr);
const struct sockaddr_in6* sockaddr_in6_cast(const struct sockaddr* addr);
struct sockaddr_in6 getLocalAddr(int sockfd);
struct sockaddr_in6 getPeerAddr(int sockfd);
bool isSelfConnect(int sockfd);
```
