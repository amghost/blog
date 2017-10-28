[RFC 5681](https://tools.ietf.org/html/rfc5681)

四个主题：慢启动(slow start), 拥塞避免(congestion avoidance), 快速重传(fast retransmit), 快速恢复(fast recovery)

## 几个重要的名词

发送者拥塞窗口（Congestion Window，cwnd）
接受者接收窗口（Receiver Window，rwnd）
发送者发送窗口 = min{cwnd, rwnd}
慢启动门限（Slow Start Threshold, ssthresh）

SMSS(Sender Maximum Segment Size): 发送者可以发送的最大segment大小，通过MTU、RMSS等其他因素得到。不包含TCP/IP的头部。以太网MTU是1500字节，所以不考虑RMSS的话，SMSS一般是 1500(MTU) - 20(TCP头) - 20(IP头) = 1460，参考[这篇文章](http://blog.csdn.net/keyouan2008/article/details/5843388)

RMSS(Receiver Maximum Segment Size): 接收者准备接收的最大segment大小，在建立连接阶段通过MSS选项指定，如果没有指定则是536字节

IW(Initial Window): 初始拥塞窗口大小，在RFC 2001中，IW被设置为1个segment（Stevens提案，所以在TCP/IP详解卷1中说的是1个segment），在后续更新的RFC 5681和 RFC 3390中，有更明确的公式：
    `min (4*MSS, max (2*MSS, 4380 bytes))`
这里4380是针对一般场景下 MTU=1500，MSS=1460。IW=4380时允许发送者初始可以发送3个segment
```
If SMSS > 2190 bytes:
    IW = 2 * SMSS bytes and MUST NOT be more than 2 segments
If (SMSS > 1095 bytes) and (SMSS <= 2190 bytes):
    IW = 3 * SMSS bytes and MUST NOT be more than 3 segments
if SMSS <= 1095 bytes:
    IW = 4 * SMSS bytes and MUST NOT be more than 4 segments
```
增大IW的好处：
1. 如果IW设置为1，如果接收者执行延迟ACK（比如捎带确认机制），因为发送者无法发下一个包，所以接收者要等到延迟超时才会发ACK。如果有2个包，则接收者在收到第二个包时会马上返回ACK
2. 对于只传输小量数据的连接，比如大部分EMail、小于4K的HTTP请求，传输时间可以减少到一个RTT

增大IW的坏处：
1. 在拥塞的网络中，突发流量（Bursty Traffic）导致的路由器Drop tail（参考[知乎-车小胖的文章](https://zhuanlan.zhihu.com/p/30404184)）会使TCP面临没有必要的超时重传

## Sequence空间
### Send Sequence Space

                   1         2          3          4
              ----------|----------|----------|----------
                     SND.UNA    SND.NXT    SND.UNA
                                          +SND.WND

        1 - old sequence numbers which have been acknowledged
        2 - sequence numbers of unacknowledged data
        3 - sequence numbers allowed for new data transmission
        4 - future sequence numbers which are not yet allowed

SND.UNA: 最小的未被确认的seq
SND.NXT: 下个可用于发送的seq
SND.WND: 发送窗口大小，2阶段和3阶段之和

### Receive Sequence Space

                       1          2          3
                   ----------|----------|----------
                          RCV.NXT    RCV.NXT
                                    +RCV.WND

        1 - old sequence numbers which have been acknowledged
        2 - sequence numbers allowed for new reception
        3 - future sequence numbers which are not yet allowed

RCV.NXT: 下一个接收sequence
RCV.WND: 接收窗口大小

## 慢启动（Slow start）
采用慢启动的原因：TCP启动时并不知道网络的状态，所以需要先探测一下网络的状态，避免一下子发出太多的数据包导致拥塞更加严重（如果有的话）

ssthresh 一般初始设置得比较大，cwnd < ssthresh 时启用慢启动算法。
慢启动算法每次收到一个ACK时就会增加cwnd，所以cwnd增长是指数级的，比如 1个包发出去收到ack，1->2，然后2个包发出去收到2个ack，2->4.
每次收到ACK，TCP对cwnd最多增加SMSS，**一般的TCP实现都是直接增加SMSS**，但是RFC 5681建议 `cwnd += min(N, SMSS)`，其中 N 为本次ACK包确认收到的数据量。由于ACK存在 ACK Division（即同一个segment的确认分成多个ack）否则若采用直接增加SMSS，会使cwnd被不恰当地放大（inappropriately inflate），采用这种方法可以更精准地控制以及增强系统鲁棒性。

## 拥塞避免（Congestion avoidence）
cwnd > ssthresh 时，进入拥塞避免阶段。该阶段中每个RTT，cwnd只增长一个segment(不超过SMSS)，直到发生网络拥塞。

当发生丢包时（通过重传超时检测到），减少 `ssthresh = max (FlightSize / 2, 2*SMSS)`。这里 `FlightSize`是发出去还在网络中的数据量。有的实现不用FlightSize而直接用cwnd，这样是不太合适的。
> Implementation Note: An easy mistake to make is to simply use cwnd, rather than FlightSize, which in some implementations may incidentally increase well beyond rwnd.

此外，cwnd被重置为 LW（loss window），其值为1（注意不是上文说的 IW），重新开始慢启动流程

## 快重传（Fast retransmit）和 快恢复（Fast recovery）
接收者接收到乱序的segment时应该立即返回一个ack（duplicate ack），ack序号为期望按序接收到的seq
从发送者的角度，造成该现象有多种可能：
1. 丢包
2. 网络中包乱序
3. ack或者segment被复制

发送者在收到3个duplicate ack之后，立即重传指定的segment，不等待重传超时。
此时不执行Slow Start，而是执行 Fast Recovery，因为接收到duplicate ack不仅意味着有segment丢了，**同时也意味着有相应数量的segment已经被接收者接收到且被缓存起来了**（接收者只有接收到乱序的segment才会发duplicate ack），这些segments已经离开网络，不消耗网络资源了（不考虑在网络中被复制的情况下）

算法步骤：
1. 在第一和第二个duplicate ack到来时，各自发送一个未发送的segment，此时 FlightSize <= cwnd + 2*SMSS。但是这两个segment被确认时，不修改cwnd。如果发送者启用了SACK，除非duplicate ack包含SACK，否则不能发segment
2. 收到第三个duplicate执行`ssthresh = max (FlightSize / 2, 2*SMSS)`，减少ssthresh
3. 执行`cwnd = ssthresh + 3*SMSS`。如果后续还继续收到duplicate ack，每收到一个，`cwnd += SMSS`，对应一个segment已经被接收且buffered
3. 当收到duplicate ack对应的segment的ack，执行 `cwnd = ssthresh`，收缩cwnd