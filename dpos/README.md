# DPoS股份授权证明算法概述

DPoS，即Delegated Proof of Stake，译为股份授权证明。
最早于2013年由比特股Bitshares提出，目的为解决PoW和PoS机制的不足。

### PoW及PoS的缺陷以及DPoS的提出

PoW机制纯粹依赖算力，导致专业挖矿群体与社区完全分隔，矿池的巨大算力形成另外的中心。
这与比特币的去中心化思想冲突。
PoS虽然考虑了PoW的不足，但会导致首富账户的权力更大，有能力支配记账权。

比特股是最早采用DPoS机制的加密货币，期望通过引入技术民主层来减少中心化的负面影响。

### DPoS的原理

DPoS引入了见证人的概念，见证人可以生成区块，每个持股人都可以投票选举见证人。
得到总票数前N（通常为101）的候选者，可以当选见证人。
见证人的候选者名单每个维护周期（1天）更新一次。

见证人随机排列，每个见证人有2秒的权限时间生成区块。
如果见证人在给定时间内无法生成区块，区块生成权限交给下一个时间片对应的见证人。
DPoS这种设计使得区块生成更快捷，也更节能。

投票选出的N个见证人，可以视为N个矿池。
如果它们提供的算力不稳定、宕机、或者作恶，持股人可以随时投票更换见证人。

### 参考文章

* [股份授权证明机制（DPOS）](http://blog.sina.com.cn/s/blog_6ab284e40102v0nw.html)
* [缺失的白皮书：DPOS共识算法工作原理及鲁棒性根源分析](https://www.leiphone.com/news/201706/JfsBmaf6Y0ZtV11R.html)
* [比特股环境搭建](http://www.blockchainbrother.com/article/53)
* [bitshares-core](https://github.com/bitshares/bitshares-core)
* [浅析 BitShares 2.0 的引荐机制及终身会员的各种玩法](http://8btc.com/thread-38026-1-1.html)
* [股份授权证明机制简介（DPOS Consensus Algorithm）](https://www.jianshu.com/p/3d9c751b2ac8)

### 后记

待续。











