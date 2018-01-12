# PoS权益证明算法原理及其在点点币、黑币中的实现

PoS，即Proof of Stake，译为权益证明。
无论PoW或PoS，均可以理解为“谁有资格写区块链”的问题。
PoW通过算力证明自己有资格写区块链，而PoS则是通过拥有的币龄来证明自己有资格写区块链。

### PoW的优势和弊端

PoW，优势为可靠，使用广泛，是经历了充分的实践检验的公有链共识算法。
但其缺点也较为明显：
* 1、消耗了太多额外算力，即大量能源。
* 2、资本大量投资矿机，导致算力中心化，有51%攻击的安全隐患。

### PoS的提出和点点币

第一个基于PoS的虚拟币是点点币。
鉴于PoW的缺陷，2012年Sunny King提出了PoS，并基于PoW和PoS的混合机制发布了点点币PPCoin。
前期采用PoW挖矿开采和分配货币，以保证公平。后期采用PoS机制，保障网络安全，即拥有51%货币难度更大，从而防止51%攻击。

PoS核心概念为币龄，即持有货币的时间。例如有10个币、持有90天，即拥有900币天的币龄。
另外使用币，即意味着币龄的销毁。
在PoS中有一种特殊的交易称为利息币，即持有人可以消耗币龄获得利息，同时获得为网络产生区块、以及PoS造币的优先权。

### 点点币的PoS实现原理

点点币的PoS证明计算公式为：

proofhash < 币龄x目标值

展开如下：

hash(nStakeModifier + txPrev.block.nTime + txPrev.offset + txPrev.nTime + txPrev.vout.n + nTime) < bnTarget x bnCoinDayWeight

* 其中proofhash，对应一组数据的哈希值，即hash(nStakeModifier + txPrev.block.nTime + txPrev.offset + txPrev.nTime + txPrev.vout.n + nTime)。
* 币龄即bnCoinDayWeight，即币天，即持有的币数乘以持有币的天数，此处天数最大值为90天。
* 目标值，即bnTarget，用于衡量PoS挖矿难度。目标值与难度成反比，目标值越大、难度越小；反之亦然。

由公式可见，持有的币天越大，挖到区块的机会越大。

peercoin-0.6.1ppc中PoS证明计算代码如下：

```c++
bool CheckStakeKernelHash(unsigned int nBits, const CBlockHeader& blockFrom, unsigned int nTxPrevOffset, const CTransaction& txPrev, const COutPoint& prevout, unsigned int nTimeTx, uint256& hashProofOfStake, bool fPrintProofOfStake)
{
    if (nTimeTx < txPrev.nTime)  // Transaction timestamp violation
        return error("CheckStakeKernelHash() : nTime violation");

    unsigned int nTimeBlockFrom = blockFrom.GetBlockTime();
    if (nTimeBlockFrom + nStakeMinAge > nTimeTx) // Min age requirement
        return error("CheckStakeKernelHash() : min age violation");

	//目标值使用nBits
    CBigNum bnTargetPerCoinDay;
    bnTargetPerCoinDay.SetCompact(nBits);
    int64 nValueIn = txPrev.vout[prevout.n].nValue;
    // v0.3 protocol kernel hash weight starts from 0 at the 30-day min age
    // this change increases active coins participating the hash and helps
    // to secure the network when proof-of-stake difficulty is low
    int64 nTimeWeight = min((int64)nTimeTx - txPrev.nTime, (int64)STAKE_MAX_AGE) - (IsProtocolV03(nTimeTx)? nStakeMinAge : 0);
	//计算币龄，STAKE_MAX_AGE为90天
    CBigNum bnCoinDayWeight = CBigNum(nValueIn) * nTimeWeight / COIN / (24 * 60 * 60);
    // Calculate hash
    CDataStream ss(SER_GETHASH, 0);
	//权重修正因子
    uint64 nStakeModifier = 0;
    int nStakeModifierHeight = 0;
    int64 nStakeModifierTime = 0;
    if (IsProtocolV03(nTimeTx))  // v0.3 protocol
    {
        if (!GetKernelStakeModifier(blockFrom.GetHash(), nTimeTx, nStakeModifier, nStakeModifierHeight, nStakeModifierTime, fPrintProofOfStake))
            return false;
        ss << nStakeModifier;
    }
    else // v0.2 protocol
    {
        ss << nBits;
    }

	//计算proofhash
	//即计算hash(nStakeModifier + txPrev.block.nTime + txPrev.offset + txPrev.nTime + txPrev.vout.n + nTime)
    ss << nTimeBlockFrom << nTxPrevOffset << txPrev.nTime << prevout.n << nTimeTx;
    hashProofOfStake = Hash(ss.begin(), ss.end());
    if (fPrintProofOfStake)
    {
        if (IsProtocolV03(nTimeTx))
            printf("CheckStakeKernelHash() : using modifier 0x%016" PRI64x" at height=%d timestamp=%s for block from height=%d timestamp=%s\n",
                nStakeModifier, nStakeModifierHeight,
                DateTimeStrFormat(nStakeModifierTime).c_str(),
                mapBlockIndex[blockFrom.GetHash()]->nHeight,
                DateTimeStrFormat(blockFrom.GetBlockTime()).c_str());
        printf("CheckStakeKernelHash() : check protocol=%s modifier=0x%016" PRI64x" nTimeBlockFrom=%u nTxPrevOffset=%u nTimeTxPrev=%u nPrevout=%u nTimeTx=%u hashProof=%s\n",
            IsProtocolV05(nTimeTx)? "0.5" : (IsProtocolV03(nTimeTx)? "0.3" : "0.2"),
            IsProtocolV03(nTimeTx)? nStakeModifier : (uint64) nBits,
            nTimeBlockFrom, nTxPrevOffset, txPrev.nTime, prevout.n, nTimeTx,
            hashProofOfStake.ToString().c_str());
    }

    // Now check if proof-of-stake hash meets target protocol
	//判断是否满足proofhash < 币龄x目标值
    if (CBigNum(hashProofOfStake) > bnCoinDayWeight * bnTargetPerCoinDay)
        return false;
    if (fDebug && !fPrintProofOfStake)
    {
        if (IsProtocolV03(nTimeTx))
            printf("CheckStakeKernelHash() : using modifier 0x%016" PRI64x" at height=%d timestamp=%s for block from height=%d timestamp=%s\n",
                nStakeModifier, nStakeModifierHeight, 
                DateTimeStrFormat(nStakeModifierTime).c_str(),
                mapBlockIndex[blockFrom.GetHash()]->nHeight,
                DateTimeStrFormat(blockFrom.GetBlockTime()).c_str());
        printf("CheckStakeKernelHash() : pass protocol=%s modifier=0x%016" PRI64x" nTimeBlockFrom=%u nTxPrevOffset=%u nTimeTxPrev=%u nPrevout=%u nTimeTx=%u hashProof=%s\n",
            IsProtocolV03(nTimeTx)? "0.3" : "0.2",
            IsProtocolV03(nTimeTx)? nStakeModifier : (uint64) nBits,
            nTimeBlockFrom, nTxPrevOffset, txPrev.nTime, prevout.n, nTimeTx,
            hashProofOfStake.ToString().c_str());
    }
    return true;
}
//代码位置src/kernel.cpp
```

### 点点币的PoS挖矿难度

点点币使用目标值来衡量挖矿难度，目标值与难度成反比，目标值越大、难度越小；反之亦然。
当前区块的目标值与前一个区块目标值、前两个区块的时间间隔有关。

计算公式如下：

当前区块目标值 = 前一个区块目标值 x (1007x10x60 + 2x前两个区块时间间隔) / (1009x10x60)

由公式可见，两个区块目标间隔时间即为10分钟。
如果前两个区块时间间隔大于10分钟，目标值会提高，即当前区块难度会降低。
反之，如果前两个区块时间间隔小于10分钟，目标值会降低，即当前区块难度会提高。

peercoin-0.6.1ppc中目标值计算代码如下：

```c++
unsigned int static GetNextTargetRequired(const CBlockIndex* pindexLast, bool fProofOfStake)
{
    if (pindexLast == NULL)
        return bnProofOfWorkLimit.GetCompact(); // genesis block

    const CBlockIndex* pindexPrev = GetLastBlockIndex(pindexLast, fProofOfStake);
    if (pindexPrev->pprev == NULL)
        return bnInitialHashTarget.GetCompact(); // first block
    const CBlockIndex* pindexPrevPrev = GetLastBlockIndex(pindexPrev->pprev, fProofOfStake);
    if (pindexPrevPrev->pprev == NULL)
        return bnInitialHashTarget.GetCompact(); // second block

    int64 nActualSpacing = pindexPrev->GetBlockTime() - pindexPrevPrev->GetBlockTime();

    // ppcoin: target change every block
    // ppcoin: retarget with exponential moving toward target spacing
    CBigNum bnNew;
    bnNew.SetCompact(pindexPrev->nBits);
	//STAKE_TARGET_SPACING为10分钟，即10 * 60
	//两个区块目标间隔时间即为10分钟
    int64 nTargetSpacing = fProofOfStake? STAKE_TARGET_SPACING : min(nTargetSpacingWorkMax, (int64) STAKE_TARGET_SPACING * (1 + pindexLast->nHeight - pindexPrev->nHeight));
	//nTargetTimespan为1周，即7 * 24 * 60 * 60
	//nInterval为1008，即区块间隔为10分钟时，1周产生1008个区块
    int64 nInterval = nTargetTimespan / nTargetSpacing;
	//计算当前区块目标值
    bnNew *= ((nInterval - 1) * nTargetSpacing + nActualSpacing + nActualSpacing);
    bnNew /= ((nInterval + 1) * nTargetSpacing);

    if (bnNew > bnProofOfWorkLimit)
        bnNew = bnProofOfWorkLimit;

    return bnNew.GetCompact();
}
//代码位置src/kernel.cpp
```

### PoS 2.0的提出和黑币

为了进一步巩固PoS的安全，2014年rat4（Pavel Vasin）提出了PoS 2.0，并发布了黑币。
黑币前5000个块，为纯PoW阶段；第5001个块到第10000个块为PoW与PoS并存阶段，从第10001个块及以后为纯PoS阶段。
黑币首创快速挖矿+低股息发行模式，发行阶段采用POW方式，通过算法改进在短时间内无法制造出专用的GPU和AISC矿机，解决分配不公平的问题。

PoS2.0相比PoS的改进：

* 1、将币龄从等式中拿掉。新系统采用如下公式计算权益证明：

　　proofhash < 币数x目标值

点点币中，部分节点平时保持离线，只在积累了可观的币龄以后才连线获取利息，然后再次离线。
PoS 2.0中拿掉币龄，使得积攒币龄的方法不再有效，所有节点必须更多的保持在线，以进行权益累积。
越多的节点在线进行权益累积，系统遭遇51%攻击的可能性就越低。

* 2、为了防范预先计算攻击，权益修正因子每次均改变。
* 3、改变时间戳规则，以及哈希算法改用SHA256。

### 黑币的PoS实现原理

黑币的PoS证明计算公式为：

proofhash < 币数x目标值

展开如下：

hash(nStakeModifier + txPrev.block.nTime + txPrev.nTime + txPrev.vout.hash + txPrev.vout.n + nTime) < bnTarget * nWeight

其中proofhash，对应一组数据的哈希值，即hash(nStakeModifier + txPrev.block.nTime + txPrev.nTime + txPrev.vout.hash + txPrev.vout.n + nTime)。
币数即nWeight，目标值即bnTarget。

blackcoin-1.2.4中PoS证明计算代码如下：

```c++
static bool CheckStakeKernelHashV2(CBlockIndex* pindexPrev, unsigned int nBits, unsigned int nTimeBlockFrom, const CTransaction& txPrev, const COutPoint& prevout, unsigned int nTimeTx, uint256& hashProofOfStake, uint256& targetProofOfStake, bool fPrintProofOfStake)
{
    if (nTimeTx < txPrev.nTime)  // Transaction timestamp violation
        return error("CheckStakeKernelHash() : nTime violation");

    //目标值使用nBits
    CBigNum bnTarget;
    bnTarget.SetCompact(nBits);

    //计算币数x目标值
    int64_t nValueIn = txPrev.vout[prevout.n].nValue;
    CBigNum bnWeight = CBigNum(nValueIn);
    bnTarget *= bnWeight;

    targetProofOfStake = bnTarget.getuint256();

	//权重修正因子
    uint64_t nStakeModifier = pindexPrev->nStakeModifier;
    uint256 bnStakeModifierV2 = pindexPrev->bnStakeModifierV2;
    int nStakeModifierHeight = pindexPrev->nHeight;
    int64_t nStakeModifierTime = pindexPrev->nTime;

    //计算哈希值
	//即计算hash(nStakeModifier + txPrev.block.nTime + txPrev.nTime + txPrev.vout.hash + txPrev.vout.n + nTime)
    CDataStream ss(SER_GETHASH, 0);
    if (IsProtocolV3(nTimeTx))
        ss << bnStakeModifierV2;
    else
        ss << nStakeModifier << nTimeBlockFrom;
    ss << txPrev.nTime << prevout.hash << prevout.n << nTimeTx;
    hashProofOfStake = Hash(ss.begin(), ss.end());

    if (fPrintProofOfStake)
    {
        LogPrintf("CheckStakeKernelHash() : using modifier 0x%016x at height=%d timestamp=%s for block from timestamp=%s\n",
            nStakeModifier, nStakeModifierHeight,
            DateTimeStrFormat(nStakeModifierTime),
            DateTimeStrFormat(nTimeBlockFrom));
        LogPrintf("CheckStakeKernelHash() : check modifier=0x%016x nTimeBlockFrom=%u nTimeTxPrev=%u nPrevout=%u nTimeTx=%u hashProof=%s\n",
            nStakeModifier,
            nTimeBlockFrom, txPrev.nTime, prevout.n, nTimeTx,
            hashProofOfStake.ToString());
    }

    // Now check if proof-of-stake hash meets target protocol
	//判断是否满足proofhash < 币数x目标值
    if (CBigNum(hashProofOfStake) > bnTarget)
        return false;

    if (fDebug && !fPrintProofOfStake)
    {
        LogPrintf("CheckStakeKernelHash() : using modifier 0x%016x at height=%d timestamp=%s for block from timestamp=%s\n",
            nStakeModifier, nStakeModifierHeight,
            DateTimeStrFormat(nStakeModifierTime),
            DateTimeStrFormat(nTimeBlockFrom));
        LogPrintf("CheckStakeKernelHash() : pass modifier=0x%016x nTimeBlockFrom=%u nTimeTxPrev=%u nPrevout=%u nTimeTx=%u hashProof=%s\n",
            nStakeModifier,
            nTimeBlockFrom, txPrev.nTime, prevout.n, nTimeTx,
            hashProofOfStake.ToString());
    }

    return true;
}
```

### 附录

* [点点币github](https://github.com/peercoin)
* [黑币github](https://github.com/CoinBlack/blackcoin)
* [点点币白皮书（中文版）](https://github.com/fengchunjian/ConsensusAlgorithm/blob/master/pos/%E7%82%B9%E7%82%B9%E5%B8%81%E7%99%BD%E7%9A%AE%E4%B9%A6%EF%BC%88%E4%B8%AD%E6%96%87%E7%89%88%EF%BC%89.pdf)
* [黑币PoS协议2.0版白皮书](https://github.com/fengchunjian/ConsensusAlgorithm/blob/master/pos/%E9%BB%91%E5%B8%81PoS%E5%8D%8F%E8%AE%AE2.0%E7%89%88%E7%99%BD%E7%9A%AE%E4%B9%A6.pdf)

### 后记

PoS有种种优点，但也有所缺陷。
即因为PoS并不消耗更多的算力，因此如果出现分叉，理性节点会在所有链上同时PoS挖矿。
以至于每次分叉都会形成新的山寨币，即PoS无法很好的应对分叉。