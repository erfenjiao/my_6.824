# 一 Raft elections and log handling (Lab 2A, 2B) 选举和日志处理

## 1. a pattern in the fault-tolerant systems

容错系统的一种模式

```go
  * MR replicates computation(复制计算) but relies on a single master to organize
  * GFS replicates data but relies on the master to pick primaries(选择初选)
  * VMware FT replicates service but relies on test-and-set to pick primary
all rely on a single entity(实体) to make critical decisions(关键决策)
    nice: decisions by a single entity avoid split brain
    it's easy to make a single entity always agree with itself(使单个实体与自己一致)
```

## 2. how can we make e.g. a fault-tolerant test-and-set service?

我们怎样制作容错的测试和设置服务

```go
we need replication
  how about two servers, S1 and S2
  if both are up(两者都启动), S1 is in charge(负责), forwards decisions to S2(将决策转发给 S2)
  if S2 sees S1 is down, S2 takes over(接管) as sole test-and-set server
  what could go wrong?
  network partition! (网络分区)split brain!
```

### 2.1. the problem: computers cannot distinguish "server crashed" from "network broken"

无法分辨"服务器崩溃" 和 "网络中断"

```go
  the symptom is the same: no response to a query over the network
  this difficulty seemed insurmountable(无法克服) for a long time
  seemed to require outside agent(外部代理) (a human) to decide when to switch (切   换)servers
  we'd prefer an automated scheme!(自动化方案)
```



### 2.2. the big insight for coping w/ partition: majority vote

应对分区的重要见解:多数票

```
have an odd(奇数) number of servers, e.g. 3
  agreement from a majority is required to do anything -- 2 out of 3
  if no majority, wait
  why does majority help avoid split brain?
    at most one partition can have a majority(最多一个分区可以占多数)
    breaks the symmetry we saw with just two servers
    (打破了我们只用两台服务器看到的对称性)
  note: majority is out of all servers, not just out of live ones
  note: proceed after acquiring majority(取得多数后进行)
    don't wait for more since they may be dead
  more generally(一般来说) 2f+1 can tolerate f failed servers
    since the remaining(剩余的) f+1 is a majority of 2f+1
    if more than f fail (or can't be contacted), no progress(没有进展)
  often called "quorum" systems(仲裁系统)
```

```go
a key property(关键特性) of majorities is that any two intersect(任意两个相交)
  servers in the intersection(交叉路口服务器) can convey information about previous decisions
  e.g. another Raft leader has already been elected for this term
  例如，另一个 Raft 领导者已经在这个任期内被选举了

//-------------------------------------------------------

Two partition-tolerant replication schemes were invented around 1990,
  Paxos and View-Stamped Replication(视图标记复制)
  called "consensus" or "agreement" protocols 
  称为“共识”或“协议”的协议
  in the last 15 years this technology has seen a lot of real-world use
the Raft paper(论文) is a good introduction to modern techniques
```



# 二 topic: Raft概述

```go
state machine(状态机) replication with Raft -- Lab 3 as example:
  [diagram: clients, 3 replicas, k/v layer + state, raft layer + logs]
  Raft is a library(库) included in each replica(副本)

time diagram of one client command(命令)
  [C, L, F1, F2]
  client sends Put/Get "command" to k/v layer in leader
  leader's Raft layer adds command to log
  leader sends AppendEntries RPCs to followers
  followers add command to log
  leader waits for replies from a bare majority (including itself)
  entry(条目) is "committed"(已提交的) if a majority put it in their logs
    committed means won't be forgotten even if failures
  majority -> will be seen by the next leader's vote requests(投票请求)
  leader executes command, replies to client
  leader "piggybacks" commit info(提交信息) in next AppendEntries
  followers execute entry once leader says it's committed
  一旦领导者说它已提交，追随者就会执行条目
```



## 日志

```go
why the logs?
  the service keeps the state machine state, e.g. key/value DB
    the log is an alternate representation of the same information!
    (相同状态的代替显示)
    why both?
  the log orders the commands
    to help replicas agree on a single execution order
    (帮助副本就单个执行顺序达成一致)
    to help the leader ensure followers have identical logs
    (帮助领导者确保追随者拥有相同的日志)
  the log stores tentative(暂存) commands until committed
  the log stores commands in case(以防) leader must re-send to followers
  the log stores commands persistently for replay after reboot
//
//-------------------------------------------------------
//
are the servers' logs exact replicas of each other?(彼此精确吗)
  no: some replicas may lag(滞后)
  no: we'll see that they can temporarily have different entries(条目)
  the good news:
    they'll eventually converge to be identical
	(他们最终会收敛到相同)
    the commit mechanism ensures servers only execute stable entries
	(提交机制确保服务器只执行稳定的条目)
```



## lab 2 Raft interface(接口)

```go
1. rf.Start(command) (index, term, isleader)
    Lab 3 k/v server's Put()/Get() RPC handlers call Start()
    Start() only makes sense(有意义) on the leader
    starts Raft agreement on a new log entry
    (在新的日志条目上启动 Raft 协议)
          append to leader's log
          leader sends out AppendEntries RPCs
          Start() returns w/o waiting for RPC replies
          k/v layer's Put()/Get() must wait for commit, on applyCh
		  (k/v 层的 Put()/Get() 必须等待提交，在 applyCh)
    agreement might fail if server loses leadership before committing 
      	then the command is likely lost, client must re-send
    isleader: false if this server isn't the leader, client should try another
    term: currentTerm, to help caller detect(检查) if leader is later demoted(后来被降级)
    index: log entry to watch to see if the command was committed
    (要查看的日志条目以查看命令是否已提交)
    ApplyMsg, with Index and Command(带有索引和命令)
    each peer sends an ApplyMsg on applyCh for each committed entry
    (每个对等点在 applyCh 上为每个提交的条目发送一个 ApplyMsg)
    in log order
    (按日志顺序)
    each peer's local service code executes, updates local replica state
    leader sends RPC reply to waiting client
    (领导者向等待的客户端发送RPC)
//
//--------------------------------------------------------------------
//
there are two main parts to Raft's design:
  electing a new leader
  ensuring identical logs despite failures
```



# 三 topic: leader election (Lab 2A)

## why a leader?

```go
ensures all replicas execute the same commands, in the same order
  (some designs, e.g. Paxos, don't have a leader)
```



## Raft numbers the sequence of leaders

(raft编号领导者的顺序)

```go
  new leader -> new term
  a term has at most one leader; might have no leader
  the numbering helps servers follow latest leader, not superseded(被取代的) leader
```

## Raft peer 什么时候开始领导选举？

```go
when it doesn't hear from(收到) current leader for an "election timeout"
increments(增加) local currentTerm, tries to(尝试) collect votes(选票)
note(注意): this can lead to un-needed elections; that's slow but safe
note: old leader may still be alive and think it is the leader
```



## 如何保证一个任期内最多有一个leader？

```go
(Figure 2 RequestVote RPC and Rules for Servers)
  leader must get "yes" votes from a majority of servers
  each server can cast only one vote per term
    if candidate, votes for itself(为自己投票)
    if not a candidate, votes for first that asks (within Figure 2 rules)
    (投票给第一个提出要求的人)
  at most one server can get majority of votes for a given term
    -> at most one leader even if network partition
    -> election can succeed even if some servers have failed
  note: again, majority is out of all servers (not just the live servers实时服务器)
```



## how does a server learn about a newly elected leader?

```go
  the leader sends out AppendEntries heart-beats
    with the new higher term number
  only the leader sends AppendEntries
    only one leader per term
    so if you see AppendEntries with term T, you know who the leader for T is
  the heart-beats suppress(抑制) any new election
    leader must send heart-beats more often than the election timeout
```



## an election may not succeed for two reasons:



```go
* less than a majority of servers are reachable(可访问的)
* simultaneous(同时期) candidates split the vote, none gets majority(没有人获得更多数)
```



## 如果选举不成功会怎样？



```go
no heartbeats -> another timeout -> a new election for a new term
higher term takes precedence(优先), candidates for older terms quit(退出)
```



## 没有特别注意，选举往往会因为分裂投票而失败

```go
  all election timers likely to go off at around the same time
  every candidate votes for itself
  so no-one will vote for anyone else!
  so everyone will get exactly one vote, no-one will have a majority
```



## Raft 如何避免分裂投票？

```
  each server adds some randomness(随机性) to its election timeout period
  (每台服务器都会为其选举超时时间增加一些随机性)
  [diagram of times at which servers' timeouts expire(超时到期)]
  randomness breaks symmetry(对称性) among the servers
    one will choose lowest random delay
  hopefully enough time to elect before next timeout expires
  others will see new leader's AppendEntries heartbeats and 
    not become candidates
  randomized delays(随机延迟) are a common pattern in network protocols
```

## 如何选择选举超时？

```go
  * 至少有几个心跳间隔（以防网络丢失心跳）
    避免不必要的选举，这会浪费时间
  * 随机部分足够长，让一个候选人在下一次开始之前成功
  * 足够短，以便对失败做出快速反应，避免长时间停顿
  * 足够短，可以在测试人员感到不安之前进行几次重试
    测试人员要求在 5 秒或更短的时间内完成选举
```

## what if old leader isn't aware a new leader is elected?

```go
  也许老领导人没有看到选举信息
  也许老领导在少数网络分区
  新的领导者意味着大多数服务器都增加了 currentTerm
  任何一位老领导都会在 AppendEntries 回复中看到新任期并下台
  否则老领导将无法获得大多数回复
    所以老领导不会提交或执行任何新的日志条目
  因此没有裂脑
  但少数人可能会接受旧服务器的 AppendEntries
    所以日志可能会在旧期限结束时发散
```



# 四 Raft 与 Paxos：

```html
https://dl.acm.org/doi/10.1145/3293611.3331595
```



# 五 视频讲解

