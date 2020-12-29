## **调试Raft(Debugging Raft)**

不可避免地，你的Raft实现的第一轮迭代将会是有Bug的。第二轮迭代也会是这样。以及第三轮、第四轮。一般来讲，每一次迭代的Bug都会比上一轮迭代更少，而且，根据经验来看，你的大多数Bug都是因为没有认真遵循Figure 2而产生的。

当调试Raft时，通常有四个主要的Bug来源：活锁(livelocks)、不正确或不完善的RPC处理函数，未能遵守规则，以及term混淆。死锁也是一个常见问题，但是它们通常可以被调试出来，通过对所有的加锁(lock)和释放锁(unlock)记录日志，以及找出你正在占有且未释放的锁的方式。让我们依次来考虑这些问题:

### **活锁(Livelocks)**

当你的系统发生活锁，系统中的每个节点都在工作，但是节点的整体集合处于一种无法推进的状态。这在Raft里很容易发生，尤其是你没有认真地遵循Figure 2时。一种活锁场景尤其经常出现；没有leader正在被选举，或者一旦一个leader被选出来，某个其他节点又启动一轮选举，强制最近刚选出来的leader立即退位。

有很多原因会导致这种场景出现，但是有少数错误是我们看到很多学生都会犯的：

- 在当Figure 2中说你应该重置你的选举定时器时，确保选举定时器被**准确** 重置了。具体来说，你应该**仅** 重启你的选举定时器，如果 a) 你收到一个来自**当前** leader的`AppendEntries` RPC(即，如果`AppendEntries`参数中的term过期了，你就**不** 应该重置你的定时器)；b) 你正在开启一轮选举；或者 c)你给另一个peer**投** 票。

  最后一种情况在不可靠(unreliable)的网络中尤其重要，在这种网络中，follower可能拥有不同的日志；在这些情况中，最后往往是大多数server愿意投票给小部分server。当某个server让你给他投票时，如果你重置了选举定时器，这就可能会让一个日志过期的server和一个有更长日志(译者注:这里应该是指有更多更新的日志)的server有同样机会站出来。

  事实上，因为拥有足够新的日志的server太少了，那些server相当不可能以正常选举的方式被选举。如果你遵守Figure 2的规则，这些具备更新的日志的servers将不会被过期的servers的选举所打断，因此这些servers更有可能完成选举然后变成leader。

- 遵循Figure 2的指示，到了关于什么时候你应该开启一轮选举。具体来说，要知道如果你是一个candidate(即，你当前正在进行一轮选举)，但是选举定时器唤醒了，你应该启动**另一轮** 选举。这对于避免因延迟或被丢弃的RPCs而引起的系统停顿来讲很重要.

- 确保你在处理一个即将到来的RPC之前遵循"Rules for Server"里的第二条规则。第二条规则说明:

> 如果RPC请求或响应包含 term T > currentTerm ：设置 currentTerm = T, 转换为 follower（§5.1）

例如，如果你已经在当前term中投票，然后一个即将到来的`RequestVote` RPC有一个高于你的term，你应该**首先** 退位(step down)然后遵从它们的term(从而重置 `votedFor`)，接下来处理这个RPC，这会导致你进行投票！

### **不正确的RPC处理函数(Incorrect RPC handlers)**

尽管 Figure 2准确阐述了每个RPC处理函数应该做什么，但是一些细微的地方仍然容易被忽略。下面是一些我们反复看到的地方，并且你应该在你的实现中注意这些地方：

- 如果一个步骤说，“reply false”，这意味着你应该立即回复，并且不再执行接下来的任何步骤。
- 如果你收到一个带有`prevLogIndex`的`AppendEntries` RPC，这个`prevLogIndex`指向你的日志结尾，你应该像你确实拥有那个条目但是term不匹配一样来处理它(即，reply false)。
- 即使leader不再发送任何条目(entries)，对于`AppendEntries` RPC 处理函数的Check 2应该被执行。
- 最后一步(#5)的`min`是必要的，并且它需要通过最新的条目的index来计算。仅仅让负责将`lastApplied`和`commitIndex`之间的日志的内容进行应用(apply)的函数在当其达到日志尾部的时候停止是不够的。这是因为在你的日志中可能有与leader的日志不同的条目，这些条目位于leader发送给你的条目之后(leader发送给你的条目都与你日志中的条目相吻合)。因为 #3表明，只有当你有冲突的条目时，你才会截断你的日志，这些冲突的条目将不会被移除，并且，如果`leaderCommit`超过了Leader发送给你的条目，你可能会应用不正确的条目。
- 严格按照section 5.4 所描述的来实现“最新日志(up-to-date log)”检查是重要的。没有欺骗(译者注: 这句表示也没看懂)，只是检查一下长度。

### **未能遵守规则(Failure to follow The Rules)**

虽然对于每个RPC处理函数的实现，Raft论文给出了明确的说明，但是也有很多规则和不变量(invariant)的实现没有指明。这些列在Figure 2右边的 “Rules for Servers”区域。尽管其中的一些已经相当地不言自明，但是也有一些需要在设计你的应用程序的时候仔细注意而不至于违背这些规则：

- 在执行期间的任意时间点，如果`commitIndex` > `lastApplied`，你应该应用某个特定的日志条目。关键不在于你直接这么做(例如，在`AppendEntreis` RPC 处理函数中)，而在于你要确保这个应用(application)仅由一个实体来完成。具体来说，你将需要一个专用的“applier”，或者锁定这些应用，从而一些其他的程序不至于也检测到需要被应用的条目且尝试去应用。
- 确保你要么是周期性地，要么在`commmitIndex`更新之后(即，在`matchIndex`更新后)检查`commitIndex > lastApplied`。例如，如果你在向peers发送`AppendEntries`的同时检查`commitIndex`，你可能要等到下一个条目被追加到日志中后，才能应用你刚刚发送的条目出去并得到确认。
- 如果一个leader发出一个`AppendEntries` RPC，并且它被拒绝了，但是不是因为日志不一致(这种情况仅当我们的term匹配的时候才会发生)，接下来，你应该立即退位，并且不要更新`nextIndex`。如果你更新了`nextIndex`，你可能会和`nextIndex`的重置发生竞争，如果你立即被重新选举的话。
- leader不允许更新`commitIndex`到前一term的某个位置(或者说，将来的term)。因此，正如规则所说，你需要检查`log[N].term == currentTerm`。这是因为，如果一个条目不是来自当前term， Raft leaders不能确定该条目是否真正地被提交(且将来不再会被改变)。这在论文中的Figure 8中有阐述。

一个常见的困惑的地方是`nextIndex`和`matchIndex`的区别。具体来说，你可能注意到`matchIndex = nextIndex - 1`，并且没有去实现`matchIndex`。这是不安全的。尽管`nextIndex`和`matchIndex`通常是同时更新为一个类似的值(具体来说，`nextIndex = matchIndex + 1`)，但是它们服务于不同的目标。`nextIndex`作为leader给指定follower共享的前缀的**估量** 。它通常是乐观的(我们共享所有)，且仅在消极的响应上才会后移。例如，当一个leader刚被选举出来的时候，`nextIndex`被设置为日志结尾处的index。在某种程度上 ，`nextIndex`被用于性能——你只需要把这些东西(译者注：nextIndex指向的条目)发生给这个peer。

`matchIndex`被用于安全性。它是一个保守的度量标准，即leader和指定follower共享的日志前缀是多少。`matchIndex`不能被设置为一个太高的值，因为这可能会导致`commitIndex`向前移动得太远。这就是为什么`matchIndex`被初始化为-1(即，我们同意没有前缀)，且仅当follower**积极回复** 一个`AppendEntries` RPC时才会更新。

### **Term混淆(Term confusion)**

Term混淆是指，servers因来自旧的terms的RPCs而混淆。一般来说，当接收一个RPC时，这不是问题，因为Figure 2中的的规则详细叙述了当你见到一个旧的term是应该做什么。但是，Figure 2没有讨论当你收到一个旧的RPC**回复** 时应该做什么。根据经验，我们已经发现，目前要做的最简单的事情就是记录回复中的term(它可能高于你的当前term)，然后将当前term和你在原始的RPC中发送的term进行比较。如果两个term不同，丢弃回复并返回(return)。**仅当** 两个term相同时你才应该继续处理这个回复。这里你可能还可以通过一些巧妙的协议推理来做进一步的优化，但目前这种方法似乎效果还不错。而不这样做，就会走上一条漫长而曲折的充满血汗、泪水和绝望的道路。

一个相关但不完全相同的问题是，假设你在发送RPC和收到回复这段时间内的状态没有改变。一个比较好的例子是，当你收到一个PRC的响应时，设置`matchIndex = nextIndex - 1`，或者 `matchIndex = len(log)`。这是**不** 安全的，因为这两个值自从你发送RPC之后可能都已经被更新了。正确的做法是把`matchIndex`更新为来自你最初发送的RPC中的参数里的`prevLogIndex + len(entries[])`。

### **顺便提一下优化(An aside on optimizations)**

Raft论文包含了几个值得关注的可选功能。在6.824中，我们要求学生实现其中的两个：日志压缩(log compaction (section 7))以及加速日志回溯(accelerated log backtracking)(第8页的左边)。前者需要避免日志无限增长，后者对于将过期的follower快速更新很有用。



这些特性不是“core Raft”的一部分，并且，也不会收到像论文中主要一致性协议部分一样的关注。这篇论文相当彻底地介绍了日志压缩(在 Figure 13)，但是如果你读的比较随意，一些设计细节可能会被遗漏。

- 当对应用程序状态生成快照时，你需要确保应用程序状态与Raft日志中某个已知index之后的状态相对应。这意味着，应用程序要么需要和Raft通信确定快照对应的index是什么，要么Raft需要推迟应用其他的日志条目，直到快照完成。

- 这段文字没有讨论涉及到快照的server崩溃以及重新启动时的恢复协议。具体来说，如果Raft状态和快照被分别提交，一个server可能在持久化快照和持久化更新后的Raft状态之间崩溃。这是一个问题，因为在Figure 13中的step 7表明，快照覆盖的Raft日志**必须被丢弃** 。

  如果，当server重新启动时，它读取更新后的快照，但是读了过期的日志，它可能最终会应用那些已经被包含在快照中的日志条目。会发生这种情况是因为`commitIndex`和`lastIndex`没有被持久化，并且Raft也不知道这些日志条目以及被应用过了。修复这个问题的方法是在Raft中引入一个持久化状态，该状态记录Raft持久化的日志的第一个条目对应的index。这可以和被加载的快照的`lastIncludeIndex`进行比较来决定在日志开头的哪些元素要被丢弃。



加速日志回溯优化不是很明确，可能是因为作者认为它对于大多数部署来说不是必要的。从文字中我们无法准确清晰地知道，来自客户端的冲突的index和term是怎么样被leader用来决定要使用的`nextIndex`是什么。我们认为作者**可能** 想让你遵循的协议是：

- 如果一个follower在日志中没有`prevLogIndex`，它应该返回`conflictIndex=len(log)`和`conflictTerm = None`。
- 如果一个follower在日志中有`prevLogIndex`，但是term不匹配，它应该返回`conflictTerm = log[prevLogIndex].Term`，并且查询它的日志以找到第一个term等于`conflictTerm`的条目的index。
- 一旦收到一个冲突的响应，leader应该首先查询其日志以找到`conflictTerm`。如果它发现日志中的一个具有该term的条目，它应该设置`nextIndex`为日志中该term内的最后一个条目的索引后面的一个。
- 如果它没有找到该term内的一个条目，它应该设置`nextIndex=conflictIndex`。

一个不太完善的解决方案是，仅使用`conflictIndex`(且忽略`conflictTerm`)，这样简化了实现，但是后面，相较于仅发送必要的日志条目，leader将会偶尔发生更多的日志条目给follower将其更新。