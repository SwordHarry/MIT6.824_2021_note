# VM-Ft

## Lecture 4

How does VM FT handle network partitions? That is, is it possible that if the primary and the backup end up in different network partitions that the backup will become a primary too and the system will run with two primaries?

VMFT 如何解决网络分区的？具体而言，如果主从在不同网络分区中，是否会导致从变成主并且系统内同时有两个主服务器？

### 讨论

这是一篇有点久远的论文，里面的概念都比较基础，容错级别是机器层面，并没有引入什么高大上的共识算法

最近也有在看 redis 的内部实现 相关的书籍，感觉 VM-FT 解决脑裂的方式，有点类似于 redis 的 Sentinel，只不过VM-FT 的更简单一些

其本质是再起一个上帝视角的第三方服务

在 **2.3 Detecting and Responding to Failure** 中有提及

首先是，主从之间通过心跳来检测网络问题，若超时，则 detected failure

- 如果是从服务器失效，主服务器将正常运作并不再向该从备份
- 主服务器崩溃，从服务器需要执行完所有的 日志条目，才能作为主服务器运行

此时，通过引入一个主从之间共同访问的共享存储解决脑裂问题，当脑裂发生，主从均可访问共享存储，它们会对共享存储采用一个原子性的 test-and-set 操作，如果操作成功，成功的 server 变为主，若失败，则表明已经有 server 变为主，当前有故障的 server 自行销毁，并在另一处启动新server

若 server 无法访问 共享存储，会持续等待重试

比较粗暴的是，如果共享存储对外的网络崩了，VM将整体不对外工作了

