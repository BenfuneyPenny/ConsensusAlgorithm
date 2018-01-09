# 分布式一致性算法Paxos

Paxos是一种基于消息传递的分布式一致性算法，由Leslie Lamport（莱斯利·兰伯特）于1990提出。
是目前公认的解决分布式一致性问题的最有效算法之一。

### 要解决的问题及应用场景

Paxos算法要解决的问题，可以理解为：一个异步通信的分布式系统中，如何就某一个值（决议）达成一致。
而此处异步通信是指，消息在网络传输过程中存在丢失、超时、乱序现象。

其典型应用场景为：

在一个分布式系统中，如果各节点初始状态一致，而且每个节点执行相同的命令序列，那么最后就可以得到一个一致的状态。
为了保证每个节点执行相同的命令序列，即需要在每一条指令上执行一致性算法（如Paxos算法），来保证每个节点指令一致。

### 角色定义

涉及角色如下：

* Proposal：为了就某一个值达成一致而发起的提案，包括提案编号和提案的值
* Proposer：提案发起者，为了就某一个值达成一致，Proposer可以以任意速度、发起任意数量的提案，可以停止或重启。
* Acceptor：提案批准者，负责处理接收到的提案，响应、作出承诺、或批准提案。
* Learner：提案学习者，可以从Acceptor处获取已被批准的提案。

约定

Paxos需要遵循如下约定：
* 1、一个Acceptor必须批准它收到的第一个提案。
* 2、如果编号为n的提案被批准了，那么所有编号大于n提案，其值必须与编号为n的提案的值相同。

### 算法描述

阶段一：准备阶段

* 1、Proposer选择一个提案编号n，向Acceptor广播Prepare(n)请求。
* 2、Acceptor接收到Prepare(n)请求，
如果编号n大于之前已经响应的所有Prepare请求的编号，
那么将之前批准过的最大编号的提案反馈给Proposer，并承诺不再接收编号比n小的提案。
否则不予理会。

阶段二：批准阶段

* 1、Proposer收到半数以上的Acceptor响应后，
如果响应中不包含任何提案，那么将提案编号n和自己的值，作为提案发送Accept请求给Acceptor。
否则将编号n，与收到的响应中编号最大的提案的值，作为提案发送Accept请求给Acceptor。
* 2、Acceptor收到编号为n的Accept请求，
只要Acceptor尚未对编号大于n的Prepare请求做出响应，就可以批准这个提案。

### 系循环问题

如果Proposer1提出编号为n1的提案，并完成了阶段一。
与此同时Proposer2提出了编号为n2的提案，n2>n1，同样也完成了阶段一。
于是Acceptor承诺不再批准编号小于n2的提案，当Proposer1进入阶段二时，将会被忽略。
同理，此时，Proposer1可以提出编号为n3的提案，n3>n2，又会导致Proposer2的编号为n2的提案进入阶段二时被忽略。
以此类推，将进入死循环。

解决办法：

可以选择一个Proposer作为主Proposer，并约定只有主Proposer才可以提出提案。
因此，只要主Proposer可以与过半的Acceptor保持通信，那么但凡主Proposer提出的编号更高的提案，均会被批准。

### Learner

一旦Acceptor批准了某个提案，即将该提案发给所有的Learner。
为了避免大量通信，Acceptor也可以将批准的提案，发给主Learner，由主Learner分发给其他Learner。
考虑到主Learner单点问题，也可以考虑Acceptor将批准的提案，发给主Learner组，由主Learner组分发给其他Learner。

### 参考文档

* [Paxos Made Simple（中文翻译版）](https://github.com/oldratlee/translations/tree/master/paxos-made-simple)
* [微信自研生产级paxos类库PhxPaxos实现原理介绍](https://mp.weixin.qq.com/s?__biz=MzI4NDMyNTU2Mw==&mid=2247483695&idx=1&sn=91ea422913fc62579e020e941d1d059e#rd)
* [【原创】一步一步理解Paxos算法](https://mp.weixin.qq.com/s?__biz=MjM5MDg2NjIyMA==&mid=203607654&idx=1&sn=bfe71374fbca7ec5adf31bd3500ab95a&key=8ea74966bf01cfb6684dc066454e04bb5194d780db67f87b55480b52800238c2dfae323218ee8645f0c094e607ea7e6f&ascene=1&uin=MjA1MDk3Njk1&devicetype=webwx&version=70000001&pass_ticket=2ivcW%2FcENyzkz%2FGjIaPDdMzzf%2Bberd36%2FR3FYecikmo%3D)

### 后记

Paxos算法相对难以理解和实现，因此后来又出现了更容易理解和实现的Raft算法。
后续会单独总结。