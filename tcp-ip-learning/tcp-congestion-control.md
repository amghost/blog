[RFC 5681](https://tools.ietf.org/html/rfc5681)

四个主题：慢启动(slow start), 拥塞避免(congestion avoidance), 快速重传(fast retransmit), 快速恢复(fast recovery)

发送者拥塞窗口（Congestion Window，cwnd）
接受者接收窗口（Receiver Window，rwnd）
慢启动门限（Slow Start Threshold, ssthresh）

采用慢启动的原因：TCP启动时并不知道网络的状态，所以需要先探测一下网络的状态，避免一下子发出太多的数据包导致拥塞更加严重（如果有的话）