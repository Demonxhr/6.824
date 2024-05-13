# raft

## 超时选举

### **请求选举方**

当**时间超时**时触发选举。

超时时间150-350ms

1. 将自己的状态转变为候选人

2. 发送请求选举rpc请求（args = {任期，自己id，最后的日志索引，最后日志的任期}）

3. 收到rpc返回请求，进行以下判断

​    ① 判断当前任期是不是大于返回的任期（日志不是最新的问题导致的返回） 大于则直接返回

​    ② 判断当前还是不是候选人状态，不是则直接返回

​    ③ 判断当前任期是不是小于返回任期，是则将自己状态转为跟随者，其他初始化，返回

​    ④ 如果收到是同意投票，且当前任期和返回任期相同，则将收到的投票数加一。  后判断是否大于一半的服务器数，大于则成为领导者，初始化每个peer的nextIndex = LastIndex + 1和自己的matchIndex = LastIndex ，初始化其他参数，返回。



### **收到请求选举方**

1. 判断请求选举方的任期是否小于自己的任期， 小于拒绝投票并返回。

2. 判断请求选举方任期是否大于自己任期，大于则将自己转为跟随者，（任期相等直接进行下面判断）并进行以下判断

   ① 如果自己投过票且投的不是该选举人且自己任期等于该选举人任期， 拒绝投票，直接返回

   ② 如果该选举人的日志比自己旧不是uptodate，拒绝投票，直接返回

   ​	**注意：** 判断是不是uptodate的条件

   ​			Ⅰ.如果被选举人最后日志的任期大于自己的最后日志任期，则为最新

   ​			Ⅱ.如果被选举人最后日志任期等于自己最后日志任期，则判断被选举人最后日志索引是否大于			   等于自己最后日志索引，相同则为最新

   ​			Ⅲ.除了满足上面的条件，其他都不是uptodate

   ③ 如果不满足上面的 则投票给该选举人，重置选举时间，返回。

<img src=".\image\超时选举.png" alt="超时选举" style="zoom:40%;" />

## 日志复制（心跳包）

**心跳机制**实现日志复制

**注意:**对log操作时 要是深拷贝，不然外部log改变会导致 传入log参数改变

心跳时间 120ms

### leader方主动发送日志添加包和心跳包

可以主动发起添加日志，也可以等心跳同步日志，这里采用心跳同步，不再主动发起，就算是有新的日志进来，也要等心跳时间到了才能同步。（上一个版本采用的是，添加日志和心跳分离导致代码冗余）。

1. 心跳时间到了，封装心跳包args（任期、领导者id、上一个日志索引、上一个日志任期、领导者提交日志索引、日志（要深拷贝）） 判断是否需要同步日志 即领导者上一条日志索引大于等于领导者保存的节点下一个日志索引 则同步，否则同步日志为空   只做心跳验证。

2. 收到rpc返回，进行以下判断：

   ① 判断返回任期是否大于领导者任期， 大于则领导者变为跟随者返回

   ② 判断是否发生冲突，冲突看 follower的处理。

   Ⅰ 无冲突，更新其他节点的machindex和nextindex，然后从后往前遍历领导者日志条目 中间遍历所有成员的matchindex，如果matchindex >=index 表示该节点的index索引的日志已经同步，当超过一半同步 则将领导者的提交索引设为该索引。 

   Ⅱ 有冲突， 更新该peer的nextindex为冲突的位置。 

### follower对心跳包处理

1. 判断接受的参数任期是否小于自己任期， 小于直接返回。
2. 更新自己状态为follower
3. 判断自己的快照索引是否比参数日志索引大，大则更新自己的nextindex为日志最后索引加一，返回
4. 如果缺失日志，则直接返回自己最后一条的日志index+1   和上面3一样都是更新自己的nextindex让心跳包来决定更新日志。返回
5. 如果没缺失日志，但当前节点前一个日志任期不等于传入参数的上一个日志条目的任期，则向前遍历到上一个任期的条目，将nextindex设置为这个条目+1。返回
6. 如果上面条件都不符合，则同步日志、持久化
7. 如果 领导者的提交比当前节点提交的index大，更新当前节点的提交index为min(args.LeaderCommit, rf.getLastIndex())，取最小是因为该节点后面的日志可能还没同步。



### 提交到状态机

单独起一个协程来提交到状态机

时间设置为30ms判断一次

是否rf.lastApplied >= rf.commitIndex，  大于等于不提交，小于提交。





## 日志持久化

在rf节点状态发生变化时做一次持久化   最简单的一部分



## 日志压缩（快照机制）

<img src=".\image\snapshot.png" alt="snapshot" style="zoom:50%;" />

快照要保存的为：状态机的状态、lastincludeindex、lastincludeterm



日志压缩 安装快照是上层sevice的要求，即上层sevice通过调用该节点的snapshot来实现快照、日志的压缩，与leader添加日志无关，是每个peer自身的事。



在日志复制的时候判断当peer节点的nextindex小于领导者的快照index（lastincludeindex），

即rf.nextIndex[server]-1 < rf.lastIncludeIndex 

即领导者在appendEntries添加日志时，领导者lastIncludeIndex（含）前的日志都已经被压缩进了快照，所以无法发送lastIncludeIndex及其之前的日志给peer，所以这时要为该peer安装一次快照。

则起一个协程进行快照安装。