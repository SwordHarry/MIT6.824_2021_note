# Bitcoin_论文笔记

有趣的是，我看了下历年的 bitcoin 里的 lecture question，从2013年开始,基本都是鼓励学生动手实践，尝试去购买一些比特币；这期间比特币应该是升了很多很多了；老师的原意应该是让学生动手实践，但是附带的是赚钱了

# 讨论

课程从后期开始，就转向学习拜占庭环境下的分布式问题了，前中期，都是学习一些分布式的基础理论，但其基本是假定非拜占庭环境的；

私认为拜占庭问题不是我目前需要关注的，作了解即可，schedule 里的论文近几年大致脉络是稳定下来了，比如：GFS，Mapreduce，Raft，Zookeeper，Chain Replication，Spanner，Spark，COPS 等等，但是后期的拜占庭环境下的论文除了 Bitcoin 和 BlockStack 之外其余都在变动，故这里仅做一些了解

类似的比特币，解决拜占庭问题或者网络安全的文章还有：

- [SUNDR](https://pdos.csail.mit.edu/6.824/papers/li-sundr.pdf)：拜占庭环境下的公网公私钥文件系统，类似于 github 这种
- [Blockstack](https://pdos.csail.mit.edu/6.824/papers/blockstack-atc16.pdf)
- [How the Bitcoin protocol actually works](https://michaelnielsen.org/ddi/how-the-bitcoin-protocol-actually-works/)：这篇文章从一笔电子交易出发，用大白话解释了比特币的一些很浅显的概念
- [Working together to detect maliciously or mistakenly issued certificates.](https://certificate.transparency.dev/)
- [Transparent Logs for Skeptical Clients](https://research.swtch.com/tlog.pdf)
- [How CT fits into the wider Web PKI ecosystem](https://certificate.transparency.dev/howctworks/)
- [精读比特币论文](https://www.cnblogs.com/xinzhao/p/8584477.html)

其他的以后感兴趣了再了解

## Merkle Tree

在了解比特币的过程中，了解到其底层的数据结构是 Merkle Tree

相关文章：[Merkle Tree](https://zhuanlan.zhihu.com/p/103372259)；本质上是对数据做切分，对每个子数据分别做哈希，最终汇总再次做总哈希，保证数据难以被篡改

## 区块链

比特币在公网中采用区块链技术做支撑，其默认一个事实，在公网环境中，恶意节点不是普通节点的大多数

通过 Proof-of-work 工作量证明进行区块的生成，本质是做 hash 运算，直到结果满足某种特定条件，这种运算通常需要持续几分钟

### 双花问题

一笔钱被交易了两次：[区块链技术是如何解决双花(双重支付)问题的？](https://www.sohu.com/a/445018104_120060925)

一笔钱通过UTXO和时间戳以及规定每笔交易必须进行六次确认等方法，在大概率上致使比特币不会出现双花问题

### 分叉

由于各个节点各自算各自的，所以有可能出现同时出块并广播，这时其他节点会拿首先收到的块继续计算，如果出现更长的链，则短链交易作废，等于没有交易

## 总结

虽然区块链技术仍存在一定缺点，耗时长，消耗算力大；但是其去中心化分布式账本的思路催生了很多科研和应用场景，这是对以往传统中心化事务交易的冲击

# 参考

- [SUNDR](https://pdos.csail.mit.edu/6.824/papers/li-sundr.pdf)
- [Blockstack](https://pdos.csail.mit.edu/6.824/papers/blockstack-atc16.pdf)
- [How the Bitcoin protocol actually works](https://michaelnielsen.org/ddi/how-the-bitcoin-protocol-actually-works/)
- [Working together to detect maliciously or mistakenly issued certificates.](https://certificate.transparency.dev/)
- [Transparent Logs for Skeptical Clients](https://research.swtch.com/tlog.pdf)
- [How CT fits into the wider Web PKI ecosystem](https://certificate.transparency.dev/howctworks/)
- [Merkle Tree](https://zhuanlan.zhihu.com/p/103372259)
- [精读比特币论文](https://www.cnblogs.com/xinzhao/p/8584477.html)
- [Bitcoin 论文阅读](https://tanxinyu.work/bitcoin/)
- [区块链技术是如何解决双花(双重支付)问题的？](https://www.sohu.com/a/445018104_120060925)

 
