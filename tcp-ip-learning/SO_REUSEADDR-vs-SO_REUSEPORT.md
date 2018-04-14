参考stackoverflow上的问题：[Socket options SO_REUSEADDR and SO_REUSEPORT, how do they differ? Do they mean the same across all major operating systems?](https://stackoverflow.com/questions/14388706/socket-options-so-reuseaddr-and-so-reuseport-how-do-they-differ-do-they-mean-t?utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa)

每个TCP/UDP连接是由这样的五元组构成：

`{<protocol>, <src addr>, <src port>, <dest addr>, <dest port>}`

一个五元组唯一标识一个连接，两个不同的连接不能有相同的五元组，否则操作系统无法区分

其中：
1. protocol是在调用`socket`是指定的
2. src addr和src port是在调用`bind`（或者操作系统自动bind）时指定的

# SO_REUSEADDR
默认情况下，不同socket不能`bind`一对相同的src addr和src port。特别地，如果有socket在某个端口对于绑定了`0.0.0.0`，可以理解为它绑定了本机所有ip，所以其他任何socket都不能再绑定这个端口，不管使用哪个ip

`SO_REUSEADDR`的作用：
1. 同个端口下，多个socket（正常工作中的）只有在ip完全不同的情况下才会`bind`冲突。

```
SO_REUSEADDR       socketA        socketB         Result
---------------------------------------------------------------------
  ON/OFF       192.168.0.1:21   192.168.0.1:21    Error (EADDRINUSE)
  ON/OFF       192.168.0.1:21      10.0.0.1:21    OK
  ON/OFF          10.0.0.1:21   192.168.0.1:21    OK
   OFF             0.0.0.0:21   192.168.1.0:21    Error (EADDRINUSE)
   OFF         192.168.1.0:21       0.0.0.0:21    Error (EADDRINUSE)
   ON              0.0.0.0:21   192.168.1.0:21    OK
   ON          192.168.1.0:21       0.0.0.0:21    OK
  ON/OFF           0.0.0.0:21       0.0.0.0:21    Error (EADDRINUSE)
```

2. 对于 **half dead** 的socket（比如TCP协议下处于 TIME_WAIT 状态的），新的socket如果设置了SO_REUSEADDR，可以直接忽略老socket，完成绑定

注：`SO_REUSEADDR`不要求所有socket都设置，只要当前要进行`bind`的socket设置即可，之前已绑定的socket可以没有设置

# SO_REUSEPORT
`SO_REUSEPORT`是后面unix引进的实现，也就是说有些老版本的系统并不支持。Linux是在3.9版本才支持的。
其作用是：只要所有socket都设置`SO_REUSEPORT`，则可以都绑定到相同的ip和port上。注意这和`SO_REUSEADDR`是不同的，后者是指ip和port不能完全相同，可以接受wildcard address；而`SO_REUSEPORT`更强大的是支持ip和port完全相同。也正是这么强大的功能，需要所有socket都显式设置

`SO_REUSEPORT`并不能替代`SO_REUSEADDR`。若ip:port当前有 TIME_WAIT 的socket，除非它也设置了`SO_REUSEPORT`，否则新的socket必须设置 `SO_REUSEADDR` 才能够绑定到这个ip:port上

在linux3.9以上的版本，如果socket设置了`SO_REUSEPORT`对于TCP，系统会将连接建立请求分配到各个socket；对于UDP，系统会均匀分发数据包到各个socket。server可以利用多线程或多进程开启多个socket监听同一个Ip:port，获得轻便的负载均衡能力；另外，linux要求对一个socket设置`SO_REUSEPORT`的必须是属于同一个用户id，以防止 **Port hijacking**

# udp connect
UDP虽然是无连接的传输层协议，但是也可以调用`connect`函数，将这个udp socket的目的ip和目的port绑定到指定的ip:port，这样它就只能与这个ip:port互相发送和接收数据包。

用`nc`命令演示：

启动一个udp server并查看udp server，可以看到已经有一个udp socket绑定了34567端口，目前还没连接其他客户端
```
Shell Input:
    nc -u -l 34567
    netstat -p udp -n | grep 34567

Shell Output:
    udp4       0      0  *.34567                *.*
```

启动一个udp client，可以看到，udp client已经**connect**到udp server上了，但是因为server还没收到udp包，所以server没有进行connect
```
Shell Input:
    nc -u -p 23456 localhost 34567
    netstat -p udp -n | grep 34567

Shell output:
    udp4       0      0  127.0.0.1.23456        127.0.0.1.34567
    udp4       0      0  *.34567                *.*
```

udp client发送数据包给server，server收到包并进行了connect（这是nc的行为，server不一定要进行绑定，可以自行写一个udp server尝试），可以看到两个udp socket互相绑定了
```
Shell Input:
    udp client> hello
    netstat -p udp -n | grep 34567 

Shell Output:
    udp_server< hello
    udp4       0      0  127.0.0.1.23456        127.0.0.1.34567
    udp4       0      0  127.0.0.1.34567        127.0.0.1.23456
```

PS：如果server socket进行了`connect`
1. server只收发来自指定ip:port的数据包，其他地址的数据包都会被丢弃，此时server-client塌缩成peer-to-peer
2. 该socket必须只能调用`recv`和`send`，而非`recvfrom`和`sendto`，可以理解为类似TCP建立了连接
