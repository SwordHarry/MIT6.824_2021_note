# mit 6.824  2021 lab1 mapreduce

## lab1 mapreduce

实验：https://pdos.csail.mit.edu/6.824/labs/lab-mr.html

实验结果源码：https://github.com/SwordHarry/mit6.824

## 实验背景

1. mapreduce paper: https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/mapreduce-osdi04.pdf
2. 实现一个简易版的分布式 mapreduce demo，并能通过 test-mr.sh 中的所有测试

## 实验测试

1. word-count
2. indexer
3. map parallelism
4. reduce parallelism
5. job count
6. early exit
7. crash test

在 6 中 test-mr.sh 中有一个 `wait -n `的命令，mac 执行会报错，我就把它改为 `wait` 了

## 实验过程与结果

### mapreduce 论文

首先是熟读了 mapreduce 论文，读了几次，要实现一个 demo 论文中其实已经给出了概要，详见论文中的 3 implementation 的6个小点；

然后是这张架构图要了然于心：

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gskdg2ig4tj30zq0okwgf.jpg" alt="image-20210717125742830" style="zoom:50%;" />

但是其中也是有很多细节没有给出需要注意的，实验中给出了足够的提示，整理如下：

1. 可以借鉴`mrsequential.go`中的部分内容，确实，里面有 shuffle 和排序的操作值得借鉴
2. 中间文件可以命名为`mr-X-Y`的形式，其中 X 为 Map 任务编号，Y 为 reduce 任务编号；即总共会生成 M*R 个中间文件
   其次，结果文件应该命名为`mr-output-Y`，Y 为 reduce 任务编号，即生成 R 个结果文件
3. 因为处理的始终是键值对，可以用`encoding/json`包，将键值对输入到中间文件中
4. `ihash(key)`函数用于将给定的键映射到相应的 Reduce 任务中，即落到相应的中间文件；这里注意一个 reduce 任务读的是`mr-0-y、mr-1-y、mr-2-y、...`的列表中间文件，需要将列表文件读入内存，然后统一按照 key 排序和 shuffle：`shuffle (k, v) -> (k, [v...])`如果内存不足，可以使用外部排序
5. worker 有时是需要阻塞等待的，即 worker 需要等最后一个 map 任务完成，reduce 任务才能启动，即 master 可以给 worker 下发等待的命令
6. 如何判断 worker 已经崩溃？论文中给出的是 master 周期性 ping worker 做心跳检查，也可以让 master 等待一段时间后周期性地检查任务是否超时，即需要在分发每个任务前给任务打上开始 时间戳，如果超时，则将任务重新分发
7. worker 在崩溃时可能会遇到文件已经部分写入的情况，可以使用`ioutil.TempFile`创建临时文件，然后用 `os.Rename`进行原子命名
8. 记得给 master 的变量读写加互斥锁

在实验过程中，文件的操作，master 和 worker 的协调，崩溃恢复 这几个点也是很令人困扰

### mapreduce 架构

根据提示和自己的理解，生成架构如下

![image-20210717134658382](https://tva1.sinaimg.cn/large/008i3skNgy1gskdgee0yjj30um0i5gmx.jpg)



其中，master 相当于生产者，worker 相当于消费者，master 维护一个 map 任务的队列和一个 reduce 任务的队列，这些是共享临界区；

map 任务队列和 reduce 任务队列在初始化时就可以定下来了，因为输入是输入文件数组和 NReduce reduce 任务数量，则可以得出对应的 NMap map 任务数量以及中间文件，结果文件的数量和命名

每个 worker 可以询问 master 领取任务，master 根据目前的总体状态(map 还是 reduce 还是 done)，给 worker 从任务队列中抽取任务并分配给该 worker，worker 根据任务类型，执行对应的 map/reduce 任务

master 还需要记录当前完成的 map 任务数和完成的 reduce 任务数，还需要记录一个总体状态，只有等最后一个 map 任务完成了，才能切换到 reduce 任务

master 还可以记录一个 worker 的状态map，用于记录 worker 的健康情况或当前信息，只有所有 worker 都完成任务后，worker 们才可以和 master 退出

### mapreduce 流程

#### master

![image-20210717142123760](https://tva1.sinaimg.cn/large/008i3skNgy1gskd9ob31gj30ys0m0gnf.jpg)

流程图不是很规范

有等待操作是因为只剩下一个 map 任务在运行中，没有多余任务没分发时，其他 worker 需要等待

#### worker

<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gskd9m5c0dj30oq114dhb.jpg" alt="image-20210717233044694" style="zoom:50%;" />

### 关于实验

一开始就直接按着分布式，生产者消费者的思路去写，过了第一个实验 wc 之后，后面常规实验都过了，就剩下 early exit 和 crash test

#### eary exit

一开始不太理解 early exit 是什么意思，在看网上资料和直接看源码发现是检查 worker 是否会提前退出，并且其中的 shell 命令中有一句`wait -n`mac 运行报错直接导致不去 wait 了，然后导致实验一直失败，更正后成功

#### crash test

map 任务和 reduce 任务 都有可能随机退出，对于map 任务，此时文件还没打开和生成，但是对于 reduce 任务，此时结果文件已经打开，就需要用 `ioutil.TempFile`做一个临时文件，崩溃后并不会在磁盘生成结果文件

并且 master 需要感知到 worker 崩溃，论文中的做法是对每个 worker 做心跳检测，我这里的做法是给每个 任务 记录开始时间戳，每隔一段时间做 timeout 检测，如果任务超时则分配给其他 worker，此时可能会有结果文件已经生成但只写入部分数据，直接覆盖之

## 实验感想

一开始无从下手。。。

断断续续地做，大概花了一周时间，其中断断续续花了五天读论文和网上找资料理解 mapreduce。。。

真正写代码的时间只有两个下午，下手很难，但是从0到1以后就清晰了，在通过第一个 wc test 的时候，反馈还是很强烈的

不得不说，mit 不愧是 mit



