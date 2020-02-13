## kcp源码阅读学习笔记

​	author:   lihaiping1603@aliyun.com

​	date:   2019/02/08

###介绍(https://github.com/skywind3000/kcp)

摘自官方介绍

KCP是一个快速可靠协议，能以比 TCP浪费10%-20%的带宽的代价，换取平均延迟降低 30%-40%，且最大延迟降低三倍的传输效果。纯算法实现，并不负责底层协议（如UDP）的收发，需要使用者自己定义下层数据包的发送方式，以 callback的方式提供给 KCP。 连时钟都需要外部传递进来，内部不会有任何一次系统调用。

整个协议只有 ikcp.h, ikcp.c两个源文件，可以方便的集成到用户自己的协议栈中。也许你实现了一个P2P，或者某个基于 UDP的协议，而缺乏一套完善的ARQ可靠协议实现，那么简单的拷贝这两个文件到现有项目中，稍微编写两行代码，即可使用。

### 技术特效

摘自官方介绍

TCP是为流量设计的（每秒内可以传输多少KB的数据），讲究的是充分利用带宽。而 KCP是为流速设计的（单个数据包从一端发送到一端需要多少时间），以10%-20%带宽浪费的代价换取了比 TCP快30%-40%的传输速度。TCP信道是一条流速很慢，但每秒流量很大的大运河，而KCP是水流湍急的小激流。KCP有正常模式和快速模式两种，通过以下策略达到提高流速的结果：

#### RTO翻倍vs不翻倍：

TCP超时计算是RTOx2，这样连续丢三次包就变成RTOx8了，十分恐怖，而KCP启动快速模式后不x2，只是x1.5（实验证明1.5这个值相对比较好），提高了传输速度。

#### 选择性重传 vs 全部重传：

TCP丢包时会全部重传从丢的那个包开始以后的数据，KCP是选择性重传，只重传真正丢失的数据包。

#### 快速重传：

发送端发送了1,2,3,4,5几个包，然后收到远端的ACK: 1, 3, 4, 5，当收到ACK3时，KCP知道2被跳过1次，收到ACK4时，知道2被跳过了2次，此时可以认为2号丢失，不用等超时，直接重传2号包，大大改善了丢包时的传输速度。

#### 延迟ACK vs 非延迟ACK：

TCP为了充分利用带宽，延迟发送ACK（NODELAY都没用），这样超时计算会算出较大 RTT时间，延长了丢包时的判断过程。KCP的ACK是否延迟发送可以调节。

#### UNA vs ACK+UNA：

ARQ模型响应有两种，UNA（此编号前所有包已收到，如TCP）和ACK（该编号包已收到），光用UNA将导致全部重传，光用ACK则丢失成本太高，以往协议都是二选其一，而 KCP协议中，除去单独的 ACK包外，所有包都有UNA信息。

#### 非退让流控：

KCP正常模式同TCP一样使用公平退让法则，即发送窗口大小由：发送缓存大小、接收端剩余接收缓存大小、丢包退让及慢启动这四要素决定。但传送及时性要求很高的小数据时，可选择通过配置跳过后两步，仅用前两项来控制发送频率。以牺牲部分公平性及带宽利用率之代价，换取了开着BT都能流畅传输的效果。



### KCP协议源码结构图(https://github.com/lihp1603/kcp/tree/note)

这里介绍一篇写的很不错的源码笔记供参考（https://blog.csdn.net/yongkai0214/article/details/85156452），我也引用一下他当中的一幅图:

![img](https://img-blog.csdnimg.cn/20181221090526734.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lvbmdrYWkwMjE0,size_16,color_FFFFFF,t_70)

然后接下来的内容，就属于我个人对于源码的一些阅读认识和读书笔记了。(主要方便自己日后翻阅，如果错误，请联系我。)

首先我们对几个东西需要弄清楚:
	对于接收方，我们收到的数据，经过input函数解析的时候，先是放入rcv_buf中，进行排序，对于排好序的有序数据，我们会从rcv_buf再移动到rcv_queue接收队列中，然后用户调用recv函数的时候，我们就直接从rcv_queue队列中取出来，这样用户得到的数据就是有序的数据包了。
	而对于kcp->rcv_nxt;这个是接收方用于标识我们接下来待接收的数据，也就是说rcv_nxt前面的数据，我们已经有序的全部收到了。
	接收方在发送应答ack的时候，ack消息中，就会将这个rcv_nxt的值设置为ack中的una来告知发送方。
	

​	对于发送方，用户调用send函数的时候，我们会先将数据分包按序放入snd_queue中,然后在flush函数中，在发送之前，我们会再从snd_queue中根据窗口大小，取出这次能发送的新数据包，重新打包，并按序存入到snd_buf中。其中打包的时候，sn的序号是依次通过分包的时候，snd_nxt++来进行编号的，所以可以认为snd_nxt为我们打包的最右侧数据包序号。同时打包的时候，还会将这次需要发送消息的una设置为rcv_nxt。

​	其次我们来看一下发送方在收到对方网络传送过来的ack应答以后的处理，先调用input函数，然后根据ack消息中的una，我们先从snd_buf缓存数据中删除sn为una之前的所有数据包,因为una在ack中的值为对端kcp->rcv_nxt的值，其次是更新发送端的kcp的kcp_snd_una的值，所以发送端的kcp->snd_una表示的是我们在snd_buf中缓存的最小或者说最左侧的数据包序号。
​	然后接着根据发生方发送过来ack中的sn序号，我们从snd_buf中，再删除掉这部分对端收到的数据包。
​	经过上述两个步骤以后，于是snd_buf中缓存的数据就是我们没有收到ack的数据包了，这部分数据包，我们在下次调用flush的时候会根据是否超时，是否被ack跳包等情况来进行重传和发送。



看下kcp的拥塞控制机制:
	cwnd为发送端的拥塞控制窗口大小,ssthresh为拥塞窗口的阀值。

```c
如果网络情况比较好的话，我们就逐步加大拥塞窗口的大小，发送更多的数据。
//网络比较好的时候，调整拥塞窗口大小的算法
if (_itimediff(kcp->snd_una, una) > 0) {//如果我们发送数据缓存中最左侧的数据包序号>接收端确认的最左侧数据包序号
	if (kcp->cwnd < kcp->rmt_wnd) {//拥塞窗口大小<对端窗口大小
		IUINT32 mss = kcp->mss;
		if (kcp->cwnd < kcp->ssthresh) {//拥塞窗口阈值，以包为单位
			kcp->cwnd++;
			kcp->incr += mss;
		}	else {
			if (kcp->incr < mss) kcp->incr = mss;
			kcp->incr += (mss * mss) / kcp->incr + (mss / 16);
			if ((kcp->cwnd + 1) * mss <= kcp->incr) {
				kcp->cwnd++;
			}
		}
		if (kcp->cwnd > kcp->rmt_wnd) {
			kcp->cwnd = kcp->rmt_wnd;
			kcp->incr = kcp->rmt_wnd * mss;
		}
	}
}
```


​	
```c
如果因为ack跳过出现数据可能的丢包，我们将通过算法进行调整，将拥塞窗口的大小减少为发送出去数据量的一半，
即下次发送的时候，我们发送上次一半的数据量，来避免网络拥堵。
if (change) {//如果是因为ack被跳过一定次数认为的丢包的情况
	IUINT32 inflight = kcp->snd_nxt - kcp->snd_una;//计算有多个消息数量在网络传输中
	kcp->ssthresh = inflight / 2;//拥塞窗口大小阀值计算
	if (kcp->ssthresh < IKCP_THRESH_MIN)
		kcp->ssthresh = IKCP_THRESH_MIN;
	kcp->cwnd = kcp->ssthresh + resent;
	kcp->incr = kcp->cwnd * kcp->mss;
}

如果出现超时丢包重传的情况，说明网络情况已经很糟糕的，因为ack啥的都可能丢失了，
所以这个时候，我们直接将拥塞窗口大小设置为最小，大量减少下次的数据网络传输。
//超时丢包的情况，说明网络情况更加糟糕，需要进一步对拥塞窗口进行计算
if (lost) {//数据发送超时，认为丢包的情况
	kcp->ssthresh = cwnd / 2;
	if (kcp->ssthresh < IKCP_THRESH_MIN)
		kcp->ssthresh = IKCP_THRESH_MIN;
	kcp->cwnd = 1;
	kcp->incr = kcp->mss;
}
```


​	再来看一下发送方在计算rto的时候的如何计算的，根据接收方发送过来的ts时间戳信息，和kcp->current时间戳，我们调用

​		 ikcp_update_ack(kcp, _itimediff(kcp->current, ts));

函数，根据两端的消息时间戳，我们来更新rrt,rto等信息。

```c
static void ikcp_update_ack(ikcpcb *kcp, IINT32 rtt)
{
	IINT32 rto = 0;
	if (kcp->rx_srtt == 0) {
		kcp->rx_srtt = rtt;
		kcp->rx_rttval = rtt / 2;
	}	else {
		long delta = rtt - kcp->rx_srtt;//计算这次和之前的差值
		if (delta < 0) delta = -delta;
		kcp->rx_rttval = (3 * kcp->rx_rttval + delta) / 4;//权重计算
		kcp->rx_srtt = (7 * kcp->rx_srtt + rtt) / 8;
		if (kcp->rx_srtt < 1) kcp->rx_srtt = 1;
	}
	rto = kcp->rx_srtt + _imax_(kcp->interval, 4 * kcp->rx_rttval);
	kcp->rx_rto = _ibound_(kcp->rx_minrto, rto, IKCP_RTO_MAX);
}
```

源码阅读的注释可以在https://github.com/lihp1603/kcp/tree/note中看到。



