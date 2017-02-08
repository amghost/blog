socket编程中，表示网络地址的结构体有`struct sockaddr`，也有`struct sockaddr_in`。此外，还有一种：

```c 
struct addrinfo {
    int              ai_flags;
    int              ai_family;
    int              ai_socktype;
    int              ai_protocol;
    socklen_t        ai_addrlen;
    struct sockaddr *ai_addr;
    char            *ai_canonname;
    struct addrinfo *ai_next;

};
```

获取网络地址，我们常会指定IP地址，端口，类型（字节流还是报文），协议族等。使用`sockaddr`我们一般是自己一个个指定，其实还有一个函数是可以使用的：

```
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>

int getaddrinfo(const char *node, const char *service,
        const struct addrinfo *hints,
        struct addrinfo **res);

void freeaddrinfo(struct addrinfo *res);

const char *gai_strerror(int errcode);
```

详细的定义可以看[Linux Man Page](http://man7.org/linux/man-pages/man3/getaddrinfo.3.html)

这里`node`即地址，可以是IPv4下的点分十进制的地址，或者IPv6下的十六进制地址，或者就是一个网络地址，可以通过DNS解析得到真正的地址

`service`可以是端口号，也可以是诸如`http`, `ftp`这样的著名协议，具体有哪些可以在`/etc/services`查到

`hints`有点主要是一些提示，它也是一个`addrinfo`结构，所以也可以指定`ai_family`, `ai_socktype`等来提示要获取的地址的相应字段取值范围。此外还可以指定`ai_flags`, 比如文档中描述的：

> If the AI_PASSIVE flag is specified in hints.ai_flags, and node is
NULL, then the returned socket addresses will be suitable for
bind(2)ing a socket that will accept(2) connections.  The returned
socket address will contain the "wildcard address" (INADDR_ANY for
        IPv4 addresses, IN6ADDR_ANY_INIT for IPv6 address)

设置了`AI_PASSIVE`的`hints.ai_flags`在`node`为NULL时会返回一个适用于accept的地址，这个地址是一个『wildcard address』，比如在IPv4就是0.0.0.0，程序监听这个地址，则是对机器上所有IP都监听了（机器可能有多个IP地址）


`struct addrinfo`配合起`bind`,`listen`,`connect`等函数来用也很方便，因为它包含了`socklen_t`类型和`struct sockaddr`类型的成员，我们不需要再给每个字段一一赋值了，只要调用一下`getaddrinfo`即可。比如connect

```c
int rc = getaddrinfo("127.0.0.1", "12345", &hints, &svrAddr);
rc = connect(sfd, svrAddr->ai_addr, svrAddr->ai_addrlen);
```
```
```
