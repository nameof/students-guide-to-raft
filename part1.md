过去的几个月里，我担任了 MIT 的**6.824 分布式系统[1]**课程的助教。这门课在早期有一些基于 Paxos 一致性算法的实验，但是今年，我们决定将其转移到**Raft[2]**。Raft 是“以易于理解为目标而被设计出来的”，并且，我们的期望这个改变可以使学生的生活变得更加轻松。

本文中，以及一起的**Instructors’ Guide to Raft[3]**文章，记录了我们的 Raft 之旅，并且希望能对那些 Rust 协议的实现者以及想要对 Raft 内部原理进行更深入地了解的学生有用。如果你正在寻找 Paxos vs Raft 的比较，或者对 Raft 进行更多的教学分析，你应该去阅读 Instructors’ Guide to Raft 这篇文章。文章的底部包含了一个列表，列表里是经常被 6.824 学生问到的问题以及其答案。如果你遇到了未在本文主要内容中列出的问题，请去查看**Q&A[4]**。本文篇幅较长，但是文中提到的所有的点都是6.824学生(和助教)遇到的真实问题。本文值得一读。

## **背景**

在我们深入了解Raft之前，一些背景知识可能会有用。6.824过去有一系列使用**Go[5]**构建的**基于Paxos的实验[6]**。选择Go是因为它对于学生来说简单易学，还因为它很适合写并发的，分布式应用(goroutine相当便利)。在课程的四个实验中，学生构建一个容错(fault-tolerant)，分片式(shared)键值(key-value)存储。第一个实验要求他们构建一个基于一致性(consensus-based)的日志库，第二个实验在前者基础上添加一个键值存储，第三个则在多个容错集群之间对键空间(key space)分片，通过一个容错分片master处理配置变更。我们还有第四个实验，在该实验中，学生必须处理机器的故障和恢复而不管磁盘是否完好。该实验室可作为学生默认的期末项目。



今年，我们决定用Raft重写所有的实验。前三个实验都是相同的，但是第四个实验被丢弃，因为持久化和故障恢复已经内置于Raft之中。这篇文章主要讨论我们关于第一个实验的经验，因为它是和Raft最直接相关的，尽管我也会涉及关于在Raft之上构建应用的内容(正如第二个实验)。



Raft，对于你们当中那些刚刚接触到它的人来说，这个协议**网站[7]**上的文字是最好的描述:

> Raft是一个以易于理解为目标而被设计的一致性算法。它在容错和性能方面和Paxos是等效的。区别是它被分解为相对独立的子问题，并且它干净利落地解决了实际系统中需要的所有主要部分。我们希望Raft能够被更多的人使用，这些使用者将会能开发出大量的质量更高的基于一致性的系统 。

类似**这样[8]**的可视化可以给你带来关于协议的主要部分的较好的概述，并且这篇论文对于为什么各个部分是被需要的给出了较好的直观感受。如果你还没有阅读**extended Raft paper[9]**，你应该在继续阅读本文之前先去读一下，因为我假定(你们)对Raft已经比较熟悉。



与所有的分布式一致性协议一样，细节十分关键(the devil is very much in the details)。在没有故障的稳定状态下，Raft的行为很容易理解，并且可以以一种直观的方式来解释。例如，从可视化中可以简单地看出，假定没有故障，一个leader最终将被选出来，并且最终所有发送给leader的操作都会以正确的顺序被follower应用(applied)。但是，当把延迟消息、网络分区和发生故障的服务器纳入考虑范围时，每一个如果(if)、但是(but)、以及和(and)都变得至关重要。尤其是，我们看到的Bug一再地重复出现，仅仅是因为阅读论文时的误解和疏忽。这个问题不是Raft所独有的，而是会出现在所有提供正确性的复杂分布式系统之中。



## **实现Raft**

Raft的终极指南在Raft论文中的Figure 2中。这个figure指定了raft servers之间互相通信的每个RPC的行为，给出了servers必须维护的各种不变量(invariant)，以及指定了特定行为应该在什么时候发生。我们将会在文章的剩下部分详细讨论Figure 2。(Figure 2)需要严格遵守。

Figure 2定义了每个server应该做什么，在每个状态，对每个即将到来的RPC，以及特定的其他事情应该在什么时间发生(比如，什么时候在log中应用(apply)一个条目(entry))。最初，你可能尝试把Figure 2作为一种非正式的指南；你把它读了一遍，然后开始按照写代码实现，大致按照它说的去做。这么做，你很快能得到并允许一个大概能工作的Raft实现。紧接着，问题就开始出现了。



事实上，Figure 2相当地细致，并且它的每一个声明(statement)，在规范上都应该作为**MUST**，而不是**SHOULD** 。例如，每当你收到一个`AppendEntries`或`RequestVote` RPC时， 你可能会合理地重置一个peer的选举定时器，这表明其他某个peer认为它是leader，或者正在尝试变成leader。直观地，这意味着我们不应该干涉其中。但是，如果你仔细阅读了Figure 2，它说：

> 如果选举超时耗尽而没有收到来自当前leader的`AppendEntries` RPC或者对candidate给予投票：转为candidate。

这个区别是很重要的，因为前者的实现可以导致特定情况下的活性(liveness)大大降低。



### **细节的重要性(The importance of details)**

为了使讨论更加具体，让我们来考虑一个绊住了许多6.824学生的例子。Raft论文在很多地方都提及了**心跳(heartbeat) RPC**。具体来说，一个leader会不定期(每个心跳间隔内至少一次)向所有的peers发出一个`AppendEntries` RPC以阻止他们开启新一轮选举。如果leader没有新的条目要发生给特定的peer，`AppendEntries` RPC就不包含任何条目，且被当做是一个心跳(消息)。

我们的学生中很多人认为心跳消息是“特殊的(special)”；当一个peer收到一个心跳消息的时候，它应该把这个消息当做不同于一个非心跳的`AppendEntries` RPC来处理。具体来说，当收到一个心跳消息的时候，它们(译者注：指peers)简单地重置其选举定时器，然后返回成功，而没有执行任何Figure 2中要求的检查。这是**相当危险**的。通过接受RPC，follower隐式地告诉leader它们的日志和leader的日志一直到且包括`AppendEntries`参数中的`prevLogIndex`都是匹配的。一旦收到回复，leader可能就认为(不正确地)，某个条目已经被复制到servers中的大多数，然后开始提交。

另一个经常出现(通常在修复上面的问题之后)的问题是，一旦受到一个心跳，它们将会截断follower中紧随`prevLogIndex`之后的日志，并且接着将`AppendEntries`参数中的条目进行追加。这也是不正确的，我们可以再次查看Figure 2:

> 如果(if)一个已存在的条目和一个新条目冲突(相同的index但是不同的terms)，删除已存在的条目以及其之后的条目。

这里的`如果(if)`很关键。如果follower拥有leader发送的所有条目，follower**必须不(MUST NOT)** 截断它的日志。Leader发送的条目之后的任何元素**必须(MUST)** 被保留。这是因为我们可能会收到一个过期的来自leader的`AppendEntries`RPC，而且截断日志将意味着“收回(take back)”那些我们已经告诉leader自己在日志中拥有的条目。



### **参考资料**

[1] 6.824分布式系统: *[https://pdos.csail.mit.edu/6.824/](https://link.zhihu.com/?target=https%3A//pdos.csail.mit.edu/6.824/)*

[2] Raft: *[https://raft.github.io/](https://link.zhihu.com/?target=https%3A//raft.github.io/)*

[3] Instructors’ Guide to Raft: *[https://thesquareplanet.com/blog/instructors-guide-to-raft/](https://link.zhihu.com/?target=https%3A//thesquareplanet.com/blog/instructors-guide-to-raft/)*

[4] Q&A: *[https://thesquareplanet.com/blog/raft-qa/](https://link.zhihu.com/?target=https%3A//thesquareplanet.com/blog/raft-qa/)*

[5] Go: *[https://golang.org/](https://link.zhihu.com/?target=https%3A//golang.org/)*

[6] 基于Paxos的实验: *[http://nil.csail.mit.edu/6.824/2015/labs/lab-3.html](https://link.zhihu.com/?target=http%3A//nil.csail.mit.edu/6.824/2015/labs/lab-3.html)*

[7] 网站: *[https://raft.github.io/](https://link.zhihu.com/?target=https%3A//raft.github.io/)*

[8] 这样: *[http://thesecretlivesofdata.com/raft/](https://link.zhihu.com/?target=http%3A//thesecretlivesofdata.com/raft/)*

[9] extended Raft paper: [https://raft.github.io/raft.pdf](https://link.zhihu.com/?target=https%3A//raft.github.io/raft.pdf)
