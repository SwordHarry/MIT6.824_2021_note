# mit_6.824_2021_lab2A_leader_election

做完 lab2 之后回来写系列文章总结

如果说 lab1 的 mapreduce 是用来入门分布式系统课程的，那么 lab2 开始就是课程设计的真正开始

lab2 系列为 raft 分布式一致性协议算法的实现，论文  [extended Raft paper](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf) 更是要反复看，尤其是 Figure 2，以及第五章节的一些实现细节

raft 将分布式一致性共识分解为若干个子问题，lab2 系列也随之挂钩：

- leader election，领导选举(lab2A)
- log replication，日志复制(lab2B)
- safety，安全性(lab2B&2C)；2C除了持久化还有 Fig8 的错误日志处理

以上为 raft 的核心特性，除此之外，要用于生产环境，还有许多地方可以优化：

- log compaction，日志压缩-快照(lab2D)
- Cluster membership changes，集群成员变更

## lab2A:leader election

### 实验内容

实现 Raft 领导者选举和心跳（没有日志条目的`AppendEntries` RPC）。第 2A 部分的目标是选出一个单一的领导者，如果没有瘫痪，领导者继续担任领导者，如果旧领导者瘫痪或 往返 旧领导者的数据包丢失，则由新领导者接管丢失。运行`go test -run 2A -race`来测试你的 2A 代码。

### 实验提示

以下提示直接提取翻译自[6.824 lab2 raft](https://pdos.csail.mit.edu/6.824/labs/lab-raft.html)

- 必须用`-race`运行测试，即`go test -run 2A -race`。

- 按照论文的图 2。实现 RequestVote RPC，与选举相关的规则，以及与领导选举相关的状态，

- 在`raft.go` 的`Raft`结构体中添加图 2 中的领导人选举状态。您还需要定义一个结构来保存有关每个日志条目的信息（这里我定义`logEntry`表示一条日志条目）。

- 填写`RequestVoteArgs`和 `RequestVoteReply`结构。修改 `Make()`以创建一个后台 goroutine，当它有一段时间没有收到其他对等方(leader)的消息时，它将通过发送`RequestVote` RPC 来定期启动领导者选举。通过这种方式，peer 可以知道谁是leader，或者在无法和leader取得联系时，自己成为leader。实现`RequestVote()` RPC 处理程序，以便followers投票给候选者。

- 要实现心跳，请定义一个 `AppendEntries` RPC 结构（lab2A 还不会用到日志条目），并让领导者定期发送它们。编写一个  `AppendEntries` RPC处理程序方法来“重置”选举超时，这样当一个服务器已经当选时，其他服务器不会向前作为leader.

- 确保不同 peer 的选举超时不会总是同时触发，否则所有 peer 只会为自己投票，没有人会成为领导者。

- 测试者要求领导者每秒发送心跳 RPC 不超过十次。（即 100ms 发送一次心跳）

- 测试者要求你的 Raft 在旧领导者失败后的 5 秒内选举一个新领导者（如果大多数对等点仍然可以通信）。但是请记住，如果发生分裂投票（如果数据包丢失或候选人不幸选择相同的随机退避时间，可能会发生这种情况），领导者选举可能需要多轮投票。您必须选择足够短的选举超时（以及心跳间隔），即使选举需要多轮，也很可能在不到五秒的时间内完成。

  这条意思是，必须在5s 内能选出唯一的 leader，否则测试会失败

- 该论文的第 5.2 节提到了 150 到 300 毫秒范围内的选举超时。只有当领导者发送心跳的频率远高于每 150 毫秒一次时，这样的范围才有意义。因为测试器将您限制为每秒 10 次心跳，您将不得不使用比论文中的 150 到 300 毫秒大的选举超时时间，但不要太大，因为那样您可能无法在 5 秒内选举出领导者。

- 您可能会发现 Go 的 [rand](https://golang.org/pkg/math/rand/) 很有用。

- 您需要编写定期或延迟后采取行动的代码。最简单的方法是使用调用[time.Sleep()](https://golang.org/pkg/time/#Sleep)的循环创建一个 goroutine ；（请参阅`Make()` 为此目的创建的`ticker()`协程）。不要使用Go的`time.Timer`或`time.Ticker`，这是很难正确使用。

- 该[向导页](https://pdos.csail.mit.edu/6.824/labs/guidance.html)，对如何开发和调试代码的一些技巧。

- 如果您的代码无法通过测试，请再次阅读论文的图 2；领导选举的完整逻辑分布在图中的多个部分。

- 不要忘记实现`GetState()`。

- 测试人员在永久关闭实例时调用您的 Raft 的`rf.Kill()`。可以使用`rf.killed（） `检查是否被killed。您可能希望在所有循环中都这样做，以避免死 Raft 实例打印混乱的消息。

- Go RPC 只发送名称以大写字母开头的结构体字段。子结构还必须具有大写的字段名称（例如数组中的日志记录字段）。该`labgob`包会警告你这一点; 不要忽略警告。

### 实现思路

消化一下实验提示和查看raft论文 5.2 节，可以得出一些实现上的信息：

1. 首先明确 raft 算法中，只有三个角色：leader, candidate, follower；其状态机变更论文中描述得很清楚

2. 参考论文 Fig2，可以知道三个角色的结构体中的属性和需要实现的方法

3. 其中 leader 负责周期性地广播发送`AppendEntries` rpc请求，candidate 也负责周期性地广播发送`RequestVote` rpc 请求

4. 需要实现的 RPC 接口有：`AppendEntries`和`RequestVote`，follower 仅负责被动地接收rpc请求，从不主动发起请求（但其实 leader 和  candidate 也会收到其他 peer 发过来的请求，在网络发生错乱的时候）

5. 心跳超时需要在`Make`中起一个 ticker 做周期性检查，并且这里不建议用 timer，建议用 `time.Sleep()`，并且我这里基本全部周期性的实现都用 sleep

6. 有些周期性的 sleep(timeout)，里面的 timeout 是要随机的，比如心跳超时，选举超时

   leader 广播则不要随机，并且足够频繁，这里我使用 100ms；心跳超时和选举超时均是 [250, 400] ms

7. 需要实现`GetState()`，测试用

8. RPC 结构体属性均用大写开头，否则 golang 不导出

9. 多在代码中埋点`DPrint`，勤打印 leaderId 和 rf.me，找bug基本靠它了QAQ

10. 一定要把助教的 guide 反复观看几次，将助教的go-test-many.sh`拿到，有时候全pass可能是偶然现象，需要批量测试无错，才是正确的

#### 关于组织结构

不太建议全部代码都塞进`raft.go`里，从 lab2A 到  lab2D，我的文件的组织结构一直在变，因为即使在封装复用的情况下，一个`AppendEntries`的 rpc 方法实现都有将近一百行（包括 log 和注释）

可以按角色功能分结构，见仁见智

#### 关于 Figure2

![image-20211010213108332](https://tva1.sinaimg.cn/large/008i3skNgy1gvajftq7jmj60u00xyk1402.jpg)

很多资料，包括 student guide 都说，论文的图二需要反复检查，并且全部严格实现；

是的没错，但是在做实验的过程中发现，仅仅是严格实现也是不够的，因为 Fig2 透露的信息是有限的，并且很容易引人遐想。

首先，它没有指明 leader 和 candidate 的发送 rpc  请求后如何处理 reply 的逻辑，这些逻辑藏在了论文第5节中，只描述了 rpc 请求接收者，即 followers 的实现

并且有的规则适用于 all server，比如

> If commitIndex > lastApplied: increment lastApplied, apply log[lastApplied] to state machine (§5.3)
> If RPC request or response contains term T > currentTerm: set currentTerm = T, convert to follower (§5.1)

这两条适用于 all server 的处理需要放在 rpc 方法或者处理 reply 的哪里呢？

还有比如candidate 中的：

> If AppendEntries RPC received from new leader: convert to follower

这条又应该放在 `AppendEntries` 的哪里呢？

还有等等问题，可以理解到，Fig 2 为一个大纲，我们需要在不背离大纲上加入一些自己对于论文第5节的理解

### 实现细节

- 图2 的 State 全部补充到代码中，这个没什么好说的

- 两个 RPC 的 args 和 reply 各封装一个结构体，以及补充大写属性

- 我感觉 lab2A 相对头疼的地方在于周期性检查，状态机转换，具体 rpc 细节上

- 每次 rpc 发送 args 和 处理 reply 之前，需要判断自身状态是否发生了改变，如果发生了改变，这次请求就不发送或者返回体不处理，直接丢弃；即在做动作之前，检查自身状态是否发生变化或过期

  > 因为这个bug卡在 lab2C 很久，血泪QAQ

- 其他常见错误，其实可以参考其他资料，这里只说明自己遇到的头疼的地方

#### 普遍封装

对于刚刚提及的 all server 的规则和实现细节，是可以普遍处理的，则可以得出 rpc handler 和 返回体的一般处理形式

```go
// RpcHandler 指 AppendEntries 和 RequestVote，并没有实现这个方法啦
func (rf *Raft) RpcHandler(){
  rf.mu.Lock()
  defer func() {
		reply.Term = rf.currentTerm
		rf.mu.Unlock()
	}()
  
 	// ...
  
  if request的Term > currentTerm { set currentTerm = T, convert to follower }

  // ...
}

func (rf *Raft) RpcBroadcast() {
  for i = 0 -> n {
    
    if 角色发生了变化 {
      return
    }
    
    RpcSender()
  }
}

func (rf *Raft) RpcSender() {
  if 若该 rpc 返回体过期 或 角色发生了变化 {
    return
  }
  
  if request的Term > currentTerm { set currentTerm = T, convert to follower 并且 return}
  
  //  ...
}
```

#### 周期性检查

心跳超时可以使用一个`time.Time`接收

这里我直接给出我用到的时间：leader 100ms, followers 和 candidate 都是 [250, 400]ms

ticker 伪码如下

```go
func (rf *Raft) ticker() {
	for !rf.killed() {
		rf.mu.Lock()
		switch rf.state {
		case FOLLOWER:
			rf.mu.Unlock()
			time.Sleep(随机心跳超时)
			rf.mu.Lock()
			if 心跳超时 {
				rf.转换为candiate()
				rf.mu.Unlock()
        // 变成 candidate 后立即开始选举，并且选举是异步进行
        go 开始选举()
			} else {
				rf.mu.Unlock()
			}
		case LEADER:
			rf.mu.Unlock()
			// 成为 leader 后立马广播心跳
			go rf.广播心跳()
			time.Sleep(固定leader的时长)
		default:
			rf.mu.Unlock()
		}
	}
}
```

可以看到需要合理利用锁，因为是死循环，所以不能`defer`了，所以不要发生死锁了

----

leader 需要定期广播心跳伪码如下：

```go
func (rf *Raft) broadcastAppendEntries() {
	// 在广播命令时，这一轮广播的 follower
	for i := range rf.peers {
		if i == rf.me {
			continue
		}
		// 抽象来看：构建 args -> send rpc -> 处理 reply
		rf.mu.Lock()
    if 状态发生了变化() {
			// 在收到 1 的请求后变成 follower 的旧leader，就不允许再后续for 循环发出请求 2 了
			rf.mu.Unlock()
			return
		}
		// nextIndex > snapshotIndex
		// 发送 AE
		args := rf.构建rpc的args(i)
		go rf.发送rpc请求并处理reply(i, args)
		rf.mu.Unlock()
	}
}
```

leader 处理 rpc reply 的细节伪码如下：

```go
func (rf *Raft) sendAppendEntriesHandler(peerIndex int, args *AppendEntriesArgs) {
	ok := rf.发送ae请求(peerIndex, args, reply)
	if ok {
		rf.mu.Lock()
		defer rf.mu.Unlock()
		isRetry := rf.handlerAppendEntriesReply(peerIndex, args, reply)
		// If AppendEntries fails because of log inconsistency: decrement nextIndex and retry (§5.3)
    这里如果需要重试则需要重试
	}
}


// handlerAppendEntriesReply 处理 AppendEntries rpc 请求的响应体
func (rf *Raft) handlerAppendEntriesReply(peerIndex int, args *AppendEntriesArgs, reply *AppendEntriesReply) bool {
	// 过期 rpc 返回检查
  状态检查，过期检查()
  // If RPC request or response contains term T > currentTerm: set currentTerm = T, convert to follower (§5.1)
  Term检查()
	
	// 不管是否为心跳包，都需要做参数校验, lab2B
	if reply.Success {
		// ...
	}
}
```

#### 状态机转换

follower 到 candidate 已经在 ticker 中，并且只能在 ticker 中出现

而 candiate 到 leader，则是需要在 candiate 选举发起并且得到超过半数投票后变成 leader，选举过程在 Fig2 中描述得很清楚

1. 转换到 candidate，开始选举
2. 自增 currentTerm
3. 投票给自己
4. 重置选举超时
5. 广播`RequestVote`给其他peer
6. 之后，分类讨论
   1. 接收到大部分 peer 的投票同意，变成 leader
   2. 接收到新 leader 的`AppendEntries`，变成 follower
   3. 选举超时，重新选举

startElection 伪码如下：

```go
func (rf *Raft) startElection() {
	for !rf.killed() {
		rf.mu.Lock()
    // On conversion to candidate, start election
    // increment currentTerm
		// 任期号单调递增并且永远不会重复
		rf.currentTerm++
		// vote for self
		rf.votedFor = rf.me
		rf.mu.Unlock()
		// 非临界区 --------------------
		// 这段时间内，可能选举成功变为 leader，也可能接收到其他 leader 的心跳变为 follower，也可能选举超时
		// reset election timer
    // send RequestVote RPC's to all other servers
		go rf.广播投票() // 注意这里需要并发广播
		time.Sleep(随机选举超时)
		// 非临界区 --------------------
		// 选举超时后必须判断一次raft状态
		rf.mu.Lock()
		if 仍然还是candidate {
			// 选举超时，立即开始新选举
			rf.mu.Unlock()
		} else {
      // 可以退出了
			rf.mu.Unlock()
			return
		}
	}
}
```

----

candiate 在广播选举时处理 reply，并根据得到的票数判断是否成为 leader

广播选举伪码如下:

```go
func (rf *Raft) broadcastVote() {
	voteGranted := 1
	// 广播一律并发处理
	//  - 如果接收到来自其他大多数节点的选票，则进入 Leader 角色
	//  - 若接收到来自其他 Leader 的 AppendEntries RPC，则进入 Follower 角色
	//  - 若再次 Election Timeout，那么重新发起选举
	for i := range rf.peers {
    // 自己不需要发给自己吧 - _-
		if i == rf.me {
			continue
		}
		rf.mu.Lock()
    // 
		if 不是candiate了 {
			rf.mu.Unlock()
			return
		}
		args := 创建rpc请求体()
		rf.mu.Unlock()
    // 并发发送
    // 这里还涉及对 reply 的处理细节
		go func(peerIndex int, args *RequestVoteArgs) {
			ok := rf.发送选举投票请求(peerIndex, args, reply)
			if ok {
				rf.mu.Lock()
				defer rf.mu.Unlock()
				/**
				for all server:
					If RPC request or response contains term T > currentTerm:
						set currentTerm = T, convert to follower (§5.1)
				*/
				// 添加一些检查，防止过期的 rpc 返回
        状态检查()
        过期检查()

				// 这里暂时不能封装成 handler 或者传参，因为 voteGranted 是闭包
				// get the vote
				if reply.VoteGranted {
					voteGranted++
					if rf.是多数(voteGranted) {
						rf.转换成leader()
						return
					}
				}
			}
		}(i, args)
	}
}
```

这里用到了闭包的技巧，`voteGranted`指的是这一轮获得的获票数，会发生变量逃逸，而不需要绑定到 raft 结构体上，因为这一轮的获票数与上一轮并无关系

----

而 peers 接收到candidate 发来的 `RequestVote` 请求时的处理，这里只有两点需要实现，图2如是说：

> 1. Reply false if term < currentTerm (§5.1)
> 2. If votedFor is null or candidateId, and candidate’s log is at
> least as up-to-date as receiver’s log, grant vote (§5.2, §5.4)

而 up-to-date 在论文第5节里有一个自然段和其他资料均有说明，伪码如下：

```go

// RequestVote
// example RequestVote RPC handler.
//
func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
	// Your code here (2A, 2B).
	rf.mu.Lock()
	defer func() {
		reply.Term = rf.currentTerm
		rf.mu.Unlock()
	}()
// 1. Reply false if term < currentTerm (§5.1)
  term 检查()
  // for all server: If RPC request or response contains term T > currentTerm: set currentTerm = T, convert to follower (§5.1)
	if args.Term > rf.currentTerm {
		rf.转换为follower()
		rf.不投给任何人了()
	}
// 2. If votedFor is null or candidateId, and candidate’s log is at
// least as up-to-date as receiver’s log, grant vote (§5.2, §5.4)
	if rf.可以投票,up-to-date了(args) {
		// 清空状态后，还需要投票给 candidate
		// 投票后才可以重置心跳超时
		rf.重置心跳超时()
		rf.投票()
		reply.VoteGranted = true
	} else {
		reply.VoteGranted = false
	}
}
```

----

peers 接收到 leader 发来的 `AppendEntries` 请求时的处理，因为 lab2A 不涉及到日志条目，故2A也很简单

> 1. Reply false if term < currentTerm (§5.1)

```go
func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
	rf.mu.Lock()
	defer func() {
		reply.Term = rf.currentTerm
		rf.mu.Unlock()
	}()
	// Reply false if term < currentTerm (§5.1)
  if term < currentTerm {
    return false
  }

	// 2A
  // If RPC request or response contains term T > currentTerm: set currentTerm = T, convert to follower (§5.1)
	if args.Term > rf.currentTerm {
    rf.设置voteFor()不投给任何人
	}
	// If AppendEntries RPC received from new leader: convert to follower
	rf.转换为follower并重置心跳(args.Term)

  // ...lab2B, 2C, 2D
  
	// 最后 success
	reply.Success = true
}
```

这里有点自己的想法，在判断完 term 以后，可以无脑设置状态为 follower 了，因为 Fig2 中：

> If RPC request or response contains term T > currentTerm: set currentTerm = T, convert to follower (§5.1)
>
> If AppendEntries RPC received from new leader: convert to follower

结合这两条，直接无脑转 follower 是没问题的， follower 还是 follower，candidate 变为 follower，leader 发现有不比它低的 term ，也变成 follower，起码保证了 lab2A 的  leader 唯一性

### 实验结果

![image-20211010221750816](https://tva1.sinaimg.cn/large/008i3skNgy1gvaksdbshqj60o20bedgq02.jpg)

这里也要提一下，实验结果的5列数据，从左往右分别是 测试花费的时间（秒）、Raft peer的数量（通常为 3 或 5 个）、测试期间发送的 RPC 数量、RPC 消息中的总字节数 以及 Raft 的日志条目数提交数

### 感想

做完以后再回来看，真的感触良多的，当初刚开始接触 lab2A 的时候，就像刚接触 lab1 一样，是一头雾水的，从0到1总是难的，但是要敢于码出第一行代码，哪怕是一行注释，码着码着就来感觉了

mit 6.824 🐂🍺的

