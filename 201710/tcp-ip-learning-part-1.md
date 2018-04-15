# TCP/IP 协议学习 Part 1

## ISN（Initial Sequence Number）
TCP连接三次握手时，两边都会发送SYN包，此时是使用ISN作为seq（seqeuence number）。如果不再深入一点去了解，可能我们就忽略了ISN的含义以及为什么要单独把它拉出来讲，因为连接握手时的seq是不能随便取的（比如不能直接取固定值1）。

一个TCP连接是由一个四元组（源IP，源端口，目标IP，目标端口）标识的，也就说相同的四元组即**“一个“**连接，在不同的时间可能有两个连接四元组相同，我们成它们为这个四元组所标识的连接的实例或前身（incarnation）

那如果在短时间有没有可能出现两个四元组相同的连接实例？有。会发生什么？有可能旧连接的tcp包被delay了，新连接会收到旧连接的tcp包（尤其是当刚好这个旧连接tcp包正好落在接受者的接收窗口内就悲剧了）。所以我们就明白为什么主动关闭连接的一房最后会有一个TIME_WAIT状态并等待一段时间才变成CLOSED状态

[RFC 6258](https://tools.ietf.org/html/rfc6528)
>  It is interesting to note that, as a matter of fact, protection
   against stale segments from a previous incarnation of the connection
   is enforced by preventing the creation of a new incarnation of a
   previous connection before 2*MSL have passed since a segment
   corresponding to the old incarnation was last seen (where "MSL" is
   the "Maximum Segment Lifetime" [RFC0793]).  This is accomplished by
   the TIME-WAIT state and TCP's "quiet time" concept (see Appendix B of
   [RFC1323]).

但是TIME_WAIT毕竟是主动关闭连接才可能走到的状态，有些新连接被快速创建的场景却还是不能防止上面说到的问题。所以ISN选择算法登场了。

[RFC 793](https://tools.ietf.org/html/rfc793) 描述了ISN的选择算法：
> To avoid confusion we must prevent segments from one incarnation of a
  connection from being used while the same sequence numbers may still
  be present in the network from an earlier incarnation.  We want to
  assure this, even if a TCP crashes and loses all knowledge of the
  sequence numbers it has been using.  When new connections are created,
  an initial sequence number (ISN) generator is employed which selects a
  new 32 bit ISN.  The generator is bound to a (possibly fictitious) 32
  bit clock whose low order bit is incremented roughly every 4
  microseconds.  Thus, the ISN cycles approximately every 4.55 hours.
  Since we assume that segments will stay in the network no more than
  the Maximum Segment Lifetime (MSL) and that the MSL is less than 4.55
  hours we can reasonably assume that ISN's will be unique.
  
也就是说ISN是一个4字节的数，大概每4毫秒递增1，每个4.55小时会循环一次。
注意这个递增的行为，