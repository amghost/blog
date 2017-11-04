# Delayed Acknowledgment
由于TCP和IP包头部总共40个字节，如果每1个字节发一个segment，相当于41个字节只带了1个字节的数据，在WAN网络会造成阻塞问题。
一般来说，TCP接收者在接收到数据时不会立即发送ACK，而是会等到自己也有数据要发送时再发送ACK（捎带确认，piggyback）。
值得注意的是，TCP不是在接收到数据后启动计时的，而是基于系统启动时间设置定时器，所以真正的从数据接收到数据被ack这段时长在1ms到定时器超时时长（200ms或500ms）

> A TCP SHOULD implement a delayed ACK, but an ACK should not be excessively delayed; in particular,the delay MUST be less than 0.5 seconds, and in a stream of full-sized segments there SHOULD be an ACK for at least every second segment.

超时时间不能超过500ms。如果发送者发送了一串full-sized的segments，则每两个segment要发送一个ACK。

# Nagle算法
1个TCP连接只会有一个在外部的未被确认的segment，每收到一个ACK才会再发送一个segment。

Nagle算法和Delayed Ack会造成冲突。发送者发送一个包时，接收者执行Delayed Ack等待超时或下一个full-sized segment，而发送者这边由于mei没有收到Ack故也不会发新的包。两边最终都要等到delayed ack超时才能进行下去。

RFC规定TCP协议栈应该支持Nagle算法，但是需要提供给上层程序来禁止它，可以使用 `TCP_NODELAY` 来禁用Nagle算法