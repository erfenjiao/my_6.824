# Abstract

Raft 是一种共识算法,用于管理复制的日志

> 共识算法本质上是复制状态机的实现,用于解决分布式系统中的各种容错问题.
>
> 通常使用每个服务器中存在的复制日志(命令序列)来实现状态机
>
> 命令序列:?
>
> 状态机状态:共识模块的工作是确保复制的日志在整个群集中保持一致。因此，状态机是确定性的，即每个状态机计算相同的状态和相同的输出序列。

**为了提高可理解性** , Raft 

1. 分离了共识的关键要素,如领导者选举,日志复制和安全,它执行了更强的一致性,以减少必须考虑的状态数量.

> 领导者选举:Raft通过首先选举一位杰出的领导者，然后赋予领导者完全责任来管理复制日志来实现共识。
>
> 复制日志:?

2. 减少状态空间(相对于Paxos,Raft减少了非确定性的程度和服务器之间的不一致的方式)

Raft还包括一个改变及群成员的新机制,该机志用重叠的主体保证安全

>  Raft also includes a new mechanism for changing the cluster membership, which uses overlapping majorities to guarantee safety.

# 1. Introdution

Raft在很多方面和现存的consensus algorithm类似，但是它有以下这些独特的特性：

- Strong leader：Raft比其他consensus algorithm使用了更强形式的leadership。比如，log entry只能从leader流向其他server。这简化了对于replicated log的管理并且使Raft更加易于理解。
- Leader election：Raft使用随机的时钟来选举leader。这只是在原来所有的consensus algorithm都需要的heartbeats的基础上增加了小小的一点东西，但是却简单快速地解决了冲突。
- Membership changes(成员变化)：Raft通过一种新的joint consensus的方法来实现server集合的改变，其中两个不同配置下的majority在过度阶段会发生重合。这能让集群在配置改变时也能继续正常运行。



# 2. Replicated state machine

![复制的状态机架构](/home/erfenjiao/Pictures/6.824/复制的状态机架构.jpg)

> Figure 1: Replicated state machine architecture. 
>
> The consensus algorithm manages a replicated log containing state machine commands (状态机命令)from clients. The state machines process identical sequences of commands(相同的命令序列) from the logs, so they produce the same outputs

Replicated state machine是用来解决分布式系统中的各种容错问题.

> 例如,拥有单一集群领导者的大规模系统 , 如 GFS[8] , HDFS[38] 和 RAMCloud[33] , 通常使用一个单独的Replicated state machine来管理领导者的选举,并储存必须在领导者崩溃后生存的配置信息.
>
> Replicated state machine 的例子包括 Chubby'[2]  和 ZooKeeper[11]

Replicated state machines 通常由replicated log 来**实现** , 如 Figure 1.

>Abstract 中有扩展到:状态机状态:共识模块的工作是确保复制的日志在整个群集中保持一致。因此，状态机是确定性的，即每个状态机计算相同的状态和相同的输出序列。

每个服务器储存一个包含一系列命令的日志,其状态机按顺序执行这些命令.

每个日志都包含相同的命令 , 所以每个状态机处理相同的命令序列.

因此，状态机是确定性的，即每个状态机计算相同的状态和相同的输出序列.



**共识算法的工作**: 保持复制日志的一致性.

服务器上的共识模块接受来自客户端的命令,并将他们添加到日志中.它与其他服务器上的共识模块进行通信,确保即使一些服务器失败了,也能使每条日志最终都包含相同顺序的请求.

一旦命令被成功复制,每个服务器的状态机就会按照日志的顺序处理他们,并将结果返回给客户

**共识算法的特性**: 

1. They ensure safety (never returning an incorrect result) under all non-Byzantine conditions, including network delays, partitions, and packet loss, duplication, and reordering.
2. They are fully functional (完全发挥作用)(available) as long as any majority of the servers are operational and can communicate with each other and with clients. Thus, a typical cluster(集群) of five servers can tolerate the failure of any two servers. Servers are assumed(被假定) to fail by stopping(通过停止而失败); they may later recover from state on stable storage and rejoin the cluster
3. They do not depend on timing to ensure the consistency of the logs: faulty clocks and extreme message delays can, at worst, cause availability problems. (会导致可用性问题)
4. In the common case, a command can complete as soon as a majority of the cluster has responded to a single round of remote procedure calls(单一回合的远程过程调用); a minority (少数)of slow servers need not impact overall system performance.(性能)



# 3. What’s wrong with Paxos?

1. 难以理解 , 单Paxos 与 多Paxos
2. it does not provide a good foundation for building practical implementations.



# 4. Designing for understandability

两种普遍适用的技术.

1. 分解法
2. 通过减少需要考虑的状态数量来简化状态空间,使系统更加连贯,并尽可能消除不确定性



# 5 The Raft consensus algorithm

**过程**:

1. 首先选举出 一个杰出的领导者来实现共识
2. 然后让领导者全权负责管理 Replicated log 
3. 领导者接受来自客户端的条目,将其敷着到其他服务器上,并告诉服务器何时可以将日志条目应用到他们的状态机

**好处**:

拥有一个领导者可以简化对Replicated log 的管理.例如,领导者可以决定在哪里放置新的日志条目,而不需要咨询其他服务器,并且数据以一种简单的方式从领导者流向其他服务器.

一个领导者可能会失败或与其他的服务器断开连接,在这种情况下,会选出一个新的领导者

**分解**:

考虑到领导者的方法,Raft将 consensus 问题分解为三个相对独立的问题:

1. 领袖选举
2. 日志复制
3. 安全性

# 图 2 分解解读

## **State**(状态)

### Persistent state on all servers:  

(Updated on stable storage before responding to RPCs) 

> 所有服务器上的持久性状态
>
> 在响应RPC之前在稳定的储存上进行更新

**currentTerm** 

latest term server has seen (initialized to 0 on first boot, increases monotonically) 

> 被看到的最新时期服务(初始化为0 , 单调递增)

**votedFor** 

candidateId that received vote in current term (or null if none) 

>在当前任期内获得的投票

**log[]** 

log entries; each entry contains command for state machine, and term when entry was received by leader (first index is 1) 

> 日志条目 ;  每个条目都包含状态机的命令, 以及条目被领导者收到的时间(第一个索引为1)

### Volatile(易失性,不稳定) state on all servers: 

**commitIndex** (承诺指数)

index of highest log entry known to be committed (initialized to 0, increases monotonically) 

> 已知承诺的最高日志条目的索引 

**lastApplied** 

index of highest log entry applied to state machine (initialized to 0, increases monotonically) 

> 被应用到状态机的最高日志的索引

### Volatile state on leaders: 

(Reinitialized after election , 选举后重新初始化) 

**nextIndex[]** 

for each server, index of the next log entry to send to that server (initialized to leader last log index + 1) 

> 对于每个服务器,下一条日志条目的索引被发送到该服务器

**matchIndex[]** 

for each server, index of highest log entry known to be replicated on server (initialized to 0, increases monotonically) . 

> 对于每个服务器, 已知最高日志条目的索引在服务器上被复制



## AppendEntries RPC

Invoked by leader to replicate log entries (§5.3); also used as heartbeat (§5.2). 

> 被领导者调用以复制日志条目,也作为心跳使用

### Arguments 

**term**                      leader’s term 

**leaderId**                so follower can redirect clients 

>  追随者可以重定向客户端

**prevLogIndex**      index of log entry immediately preceding new ones 

> 紧接在新日志之前的日志条目的索引 

**prevLogTerm**       term of prevLogIndex entry 

**entries[]**                 log entries to store (empty for heartbeat; may send more than one for efficiency) 

**leaderCommit**      leader’s commitIndex 

### Results:

**term**        currentTerm, for leader to update itself 

> 供领导者自我更新

**success**   true if follower contained entry matching prevLogIndex and prevLogTerm 

### Receiver implementation

1. Reply false if term < currentTerm (§5.1) 
2. Reply false if log doesn’t contain an entry at prevLogIndex whose term matches prevLogTerm (§5.3) 
3. If an existing entry conflicts with a new one (same index but different terms), delete the existing entry and all that follow it (§5.3) 
4. Append any new entries not already in the log 
5. If leaderCommit > commitIndex, set commitIndex = min(leaderCommit, index of last new entry)

> 1. 如果 term < currentTerm，则回复 false (§5.1) 
> 2. 如果在prevLogIndex处不包含术语与prevLogTerm匹配的条目，则回复false(§5.3) 
> 3. 如果一个现有的条目与一个新的条目冲突（相同的索引但不同的术语），删除现有的条目和所有跟随它的条目（§5.3）。
> 4. 4. 添加日志中尚未出现的任何新条目。
> 5. 如果 leaderCommit > commitIndex，则设置 commitIndex = min(leaderCommit, 最后一个新条目的索引)

> 未完待续,,,

# 图 3

Raft guarantees that each of these properties is true at all times. The section numbers indicate where each property is discussed.

> Raft 保证这些属性中的每一项在任何时候都是真的.节号表示每个属性的讨论位置

**Election Safety:**

at most one leader can be elected in a given term. §5.2 

**Leader Append-Only**

a leader never overwrites or deletes entries in its log; it only appends new entries. §5.3 

**Log Matching**

if two logs contain an entry with the same index and term, then the logs are identical in all entries up through the given index. §5.3 

**Leader Completeness** 

if a log entry is committed in a given term, then that entry will be present in the logs of the leaders for all higher-numbered terms. §5.4 

**State Machine Safety**

 if a server has applied a log entry at a given index to its state machine, no other server will ever apply a different log entry for the same index. §5.4.3

> **选举安全：**
>
> 在一个给定的任期内，最多可以选出一位领导人。§5.2 
>
> **领导者只需追加**
>
> 领导者从不覆盖或删除其日志中的条目；它只添加新条目。§5.3 
>
> **日志匹配**
>
> 如果两个日志包含一个具有相同索引和术语的条目，那么在给定索引之前的所有条目中，日志是相同的。§5.3 
>
> **领导的完整性** 
>
> 如果一个日志条目在一个给定的术语中被提交，那么该条目将出现在所有更高编号的术语的领导者的日志中。§5.4 
>
> **状态机安全**
>
>  如果一个服务器在它的状态机上应用了一个给定索引的日志条目，那么其他服务器将永远不会为同一索引应用不同的日志条目。§5.4.3

# 5.1 Raft 基础知识

一个Raft集群包含几个服务器,五个是比较典型的数量.

在任何时候,每个服务器都处于追随者,候选者,领导者

在正常操作中 , 只有一个领导者 , 其他所有的服务器都是追随者.

**追随者**是被动的,他们自己不发出任何请求,只会对领导者和候选人的请求作出回应.如果一个追随者没有收到任何通信,他就会成为一个候选人并发起选举

**领导者**处理所有客户的请求

**候选人**,被用来选举一个新的领导者

![Raft图5](/home/erfenjiao/Pictures/6.824/Raft图5.png)



如图所示,Raft 将时间划分为任意长度的项.期限用连续的整数来编号.

每个任期以选举开始,其中一个或多个候选人试图成为领导者.如果一个候选人在选举中获胜,那么他将在剩下的任期内担任领袖.

在某些情况下,选举的结果是分裂票,在这种情况下,任期将在没有领袖的情况下结束



不同的服务器可能在不同的时间观察到术语之间的转换。

不同的服务器可能会在不同的时间观察到不同的任期，而在某些情况下，一个服务器可能不会观察到

一次选举，甚至是整个任期。

条款在Raft中充当逻辑时钟[14]，它们允许服务器检测过时的信息，如过时的领导。

每个服务器都会存储一个当前的术语编号，该编号随着时间的推移随着时间的推移单调地增加。

每当服务器进行通信时，就会交换当前术语。

每当服务器进行通信时，就会交换；如果一个服务器的当前条款

术语小于另一个，那么它就会将其当前的

术语为较大的值。如果一个候选者或领导者发现候选者或领导者发现它的术语已经过期，它立即恢复到

追随者状态。

如果一个服务器收到一个带有过时的术语的请求 号码的请求，它将拒绝该请求。
筏子服务器使用远程程序调用(RPCs)进行通信，而基本的共识算法只需要两种类型的RPC。RequestVote RPCs是由候选人发起的（第5.2节），而AppendEntries RPC则由领导者发起，用于复制日志条目并提供一种心跳形式（第5.3节）。第7节增加了第三个RPC，用于在服务器之间传输快照。
服务器之间传输快照。如果服务器没有及时收到响应，它们会重试RPC，并以并行方式发出RPC，以获得最佳性能。

# 5.2 领导者选举

Raft使用心跳机制来触发领导者选举。当服务器启动时，它们开始作为跟随者。

服务器只要收到有效的心跳信号，就一直处于追随者状态。

来自领导者或候选者的RPC。

领导者会定期发送心跳（AppendEntries RPCs，不携带任何日志条目)到所有追随者，以保持他们的权威。

如果一个跟随者在一段时期内没有收到通信，称为选举超时。

称为选举超时，那么它就认为没有可行的领导者，并开始选举，以选择一个新的领导者。



为了开始选举,追随者增加他的当前任期并过渡到候选状态,然后,他为自己投票,并向集群中的每个服务器发出RequestVote RPCs. 候选者继续处于这种状态,直到发生三种情况之一 a)他赢得了选举 b)另一个服务器确定了自己的领导地位 c) 一段时间过去了,没有赢家



# 5.3 日志复制















