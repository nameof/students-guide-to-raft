## **Raft 之上的应用(Applications on top of Raft)**

当在 Raft 之上构建一个服务时(比如第二个 6.824 Raft 实验中的 key/value 存储)，服务和 Raft 日志间的交互很难保证正确。这部分详细讲述了在构建你的应用程序时，在开发过程中可能会对你有用的一些方面。

### **应用客户端操作(Applying client operations)**

你可能困惑于如何以复制日志的形式实现一个应用程序。你可能会这样开始，让你的服务，一旦收到一个客户端请求时，就把那个请求发送给 leader，等待 Raft 应用(apply)，处理客户端请求的操作，然后返回给客户端。这在一个单客户端系统中是没问题的，但是对于并发客户端就无法工作了。



相反，服务应该被构造为一个**状态机(state machine)** ，在这个状态机中，客户端操作将状态机的状态从一个状态转换到另一个状态。 你应该有一个循环，该循环一次仅接受一个客户端操作(在所有 servers 上以相同的顺序——这是 Raft 起作用的地方)，并依次应用到状态机。这个循环应该是你的代码中唯一接触到应用程序状态的地方(对应于 6.824 中的 key/value)。这意味着，你的面向客户端的 RPC 方法应该简单地把客户端的操作提交给 Raft，然后**等待** 这个操作被这个“applier loop”应用。仅当客户端的命令出现时，它才应该被执行，并且读取返回值。记住，**这包括读请求** 。



这就带来了另一个问题：你怎么知道什么时候一个客户端操作已经完成?在没有故障的情况下，这很简单——你只要等待你放到日志里的东西返回（即，传递给`apply()`）。当发生返回时，你把结果返回给客户端。但是，如果存在故障会发生什么呢？例如，当客户端最开始联系你的时候，你可能已经是 leader 了，但是另一个节点又被选举了，并且给你放到日志里的客户端请求也被丢弃了。很明显，你需要让客户端再试一次，但是你怎么知道什么时候告诉他们错误的信息？



解决这个问题的一个简单方式是去记录当你把客户端请求插入到 Raft 日志中时该请求所在 Raft 日志中的位置。一旦这个 index 的操作被发送到`apply()`，你可以基于该 index 出现的操作是否是你放在那里的操作来区分这个客户端请求是否成功。如果不是，说明发送了故障，并且应该给客户端返回一个错误。

### **重复检测(Duplicate detection)**

一旦你让客户端在遇到错误时重试操作，你就需要某种重复检测机制——如果一个客户端发送一个`APPEND`到你的 server，但是没有收到回复，然后重新把它发送给下一个 server，你的`apply()`函数需要确保`APPEND`没有执行两次。这么做，你需要某种针对每个客户端请求的唯一标识符，从而你可以识别出在过去是否已经看到，或者更重要的，应用(apply)，一个特定的操作。此外，这个状态需要是你状态机的一部分，从而使你所有的 Raft servers 排除相同的重复。



有许多种分配这类标识符的方式。 一个简单且相对高效的方式是，给每个客户端一个唯一标识符，然后让它们(指每个客户端)用一个单调递增序列数标记每个请求。如果一个客户端重发一个请求，它会重用相同的序列数。你的 server 记录它所见到的每个客户端的最新的序列数，任何已经见过(处理过)的操作简单地将其忽略。

### **令人头疼的边界情况(Hairy corner-cases)**

如果你的实现遵循上面给出的总纲，你很有可能会遇到至少两个微妙的问题，这些问题不经过认真的调试可能很难辨别出来。为了节省你的时间，下面是这些问题：



**重复出现的索引(Re-appearing indices)：** 比如说你的 Raft 库有个`Start()`方法，该方法接收一个命令(command)，然后返回这个命令在日志中的 index(因此你知道什么什么时候给客户端返回，正如上面所述)。你可能假定你永远不会看到`Start()`两次返回相同的 index，或者，至少，如果你重复看到相同的 index，第一次返回这个 index 的命令一定是失败了。事实证明，这两件事都不是真的，即使服务器没有崩溃。



考虑下面的场景，有五个 servers,S1 到 S5。最开始，S1 是 leader，并且它的日志是空的。

1. 两个客户端操作(C1 和 C2)到达 S1
2. `Start()`对 C1 返回 1，对 C2 返回 2
3. S1 向 S2 发出包含有 C1 和 C2 的`AppendEntries`，但是所有的其他信息都丢失了
4. S3 进位变 candidate
5. S1 和 S2 将不会投票给 S3，但是 S3，S4 和 S5 全都会投票，所以 S3 变成了 leader
6. 另一个客户端发起请求， C3 来到了 S3
7. S3 调用`Start()`（该方法返回 1）
8. S3 给 S1 发送了`AppendEntries`，S1 从日志中丢弃了 C1 和 C2，然后添加了 C3
9. S3 在给其他 servers 发送`AppendEntries`之前发生了故障
10. S1 进位，因为它的日志是最新的，所以它被选为了 leader
11. 另一个客户端发起请求，C4， 到达了 S1
12. S1 调用`Start()`，该方法返回 2(这也是`Start(C2)`返回的结果)
13. S1 所有的`AppendEntries`都被丢弃，然后 S2 进位
14. S1 和 S3 将不会投票给 S2，但是 S2，S4 和 S5 全都会给 S2 投票，所以 S2 变成了 leader
15. 一个客户端请求 C5 来到 S2
16. S2 调用`Start()`，该函数返回 3
17. S2 成功给所有 servers 发送了`AppendEntries`，S2 通过在下一次心跳消息中包含一个更新后的`leaderCommit = 3`向所有的 server 报告返回(译者注: 报告返回上一次 AppendEntries 里成功 commit 的 index)

因为 S2 的日志是[C1 C2 C5]，这意味着在 index 为 2 的提交的条目(且被应用于 S1 在内的所有 servers)是 C2。尽管 C4 是 S1 上返回 index 2 的最新的客户端操作。

**四种方式的死锁(The four-way deadlock):** 这都要归功于**[Steven Allen](https://link.zhihu.com/?target=http%3A//stebalien.com/)**，另一位6.824的助教。它发现了下面的可恶的四种方式的死锁，这四种方式的死锁在你构建基于Raft的应用时很容易遇到。



你的Raft代码，尽管它是结构化的，可能有一个类似`Start()`的函数，作用类似于允许应用程序添加新的命令到Raft日志。还有可能有一个循环，当`commitIndex`被更新的时候，在应用程序上为`lastApplied`和`commitIndex`之间的日志里的所有元素调用`apply()`。这些程序可能都有一个锁`a`。在你的基于Raft的应用中，你可能会在你的RPC处理函数中某个位置调用Raft的`Start()`函数，而且你在其他地方还有一些代码，每当Raft应用一个新的日志条目时，就会通知这些代码。因为这两处通信的需要(即，RPC方法需要知道什么时候放入日志的操作完成了)，它们可能都会拥有某个锁`b`。



在Go中，这四个代码片段可能看起来像这样：

```go
func (a *App) RPC(args interface{}, reply interface{}) {
// ...
i := a.raft.Start(args)
// update some data structure so that apply knows to poke us later
a.mutex.Unlock() // wait for apply to poke us
return

func (r *Raft) Start(cmd interface{}) int { 
r.mutex.Lock() 
// do things to start agreement on this new command
// store index in the log where cmd was placed 
r.mutex.Unlock() 
return index 
}

func (a *App) apply(index int, cmd interface{}) {
 a.mutex.Lock()
    switch cmd := cmd.(type) {
    case GetArgs:
      // do the get
     // see who was listening for this index
     // poke them all with the result of the operation
      // ...
    } 
    a.mutex.Unlock()    
}

func (r *Raft) AppendEntries(...){
// ...
r.mutex.Lock()
 // ...
 for r.lastApplied < r.commitIndex {
 r.lastApplied++ 
 r.app.apply(r.lastApplied, r.log[r.lastApplied]) 
 }
 // ...
  r.mutex.Unlock()
}
```

考虑现在系统处于下面的状态:

- App.RPC只拿到了一个`a.mutex`并且调用`Raft.Start`
- `Raft.Start`正在等待 `r.mutex`
- `Raft.AppendEntries`持有`r.mutex`，且刚刚调用了`App.apply`

我们现在有一个死锁，因为：

- `Raft.AppendEntries`将不会释放这个锁直到`App.apply`返回
- `App.apply`不能返回直到它得到`a.mutex`
- `a.mutex`将不会释放直到`App.RPC`返回
- `App.RPC`将不会返回直到`Raft.Start`返回
- `Raft.Start`将不会返回直到它拿到`r.mutex`
- `Raft.Start`必须等待`Raft.AppendEntries`

有很多种方式可以让你解决这个问题。最简单的一个方式就是在`App.RPC`中调用`a.raft.Start`之**后** 获取`a.mutex`。但是，这意味着在`App.RPC`有机会记录它希望被通知的事实之**前**，`App.apply`可能会被客户端操作调用，该客户端操作里，`App.RPC`刚刚调用了`Raft.Start`。另一个可能会产生更加整洁的设计是，使用一个单独地，专用的线程从`Raft`里调用`r.app.apply`。这个线程可能每次`commitIndex`被更新的时候都会被提醒，并且接下来不需要持有一个锁来应用，从而打破死锁。