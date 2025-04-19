---
title: "分布式协议总结"
description: ""
date: "2025-03-30T22:29:20+08:00"
thumbnail: ""
categories:
  - "分布式系统"
tags:
  - "分布式协议"
---
总结多种分布式协议的实现原理
<!--more-->
## 1. CAP 理论
- **Consistency**: 一致性，客户端每次从分布式系统读数据时，都是最新的准确的，或者返回 error
- **Availability**：可用性，客户端每次从分布式系统读数据时，得到的都是非 error 响应，并不保证一定是最新的数据
- **Partition tolerance**：分区容错性，分布式系统在内部因网络异常等原因导致数据同步失败或延迟时，产生不一致的 partition ，但系统仍可对外提供服务。    
CAP 理论指一个分布式系统不能同时满足 C A P 这三个特性。
#### 1.1 满足 CP
同时满足`一致性`和`分区容错性`，当系统中存在分区时，即节点上的数据存在不一致，要满足一致性则必然只能等待分区消失后再提供服务，或者全部节点返回 error，这样就不能再满足`可用性`
#### 1.2 满足 AP
同时满足`可用性`和`分区容错性`，当系统存在分区时，为保证可用性所有节点都必须返回非 error 的值，则必然存在不一致的值，导致不满足`一致性`
#### 1.3 满足 CA
同时满足`一致性`和`可用性`，则系统每个节点返回的数据都是最新准确非 error 的值，当系统存在分区时显然不能满足，因此系统不能对外提供服务，导致不满足`分区容错性`
## 2. Base 理论
Base 理论指分布式系统可以同时满足
- **B**asically Available：`基本可用`，相对于完全可用，通过延迟响应、流量削峰等手段保障功能整体正常，实现基本可用
- **S**oft **S**tate：`软状态`，指系统在没有外部输入时其整体状态也可能随时间而变化，因为最终一致性分区会随时间完成同步
- **E**ventually consistent：`最终一致性`，指系统的副本在经过一定时间的同步之后最终能够完全一致，不需要保证数据的强一致性  

可以认为 Base 理论在一致性和可用性上做了适当妥协来达成分区容错性    
## 3. 2PC 协议
向分布式系统提交数据时，为保证`一致性`引入一个 **协调者** 来协调所有节点的写入，各节点被称为 **参与者**   
过程：
1. Prepare 阶段：协调者给每个参与者发送 Prepare 消息，各参与者执行写入，但未提交事务;
2. Commit 阶段：如果协调者接收到参与者失败或超时的消息，则回滚所有参与者；如果全都成功则发送 Commit 消息给所有参与者。参与者根据消息执行回滚或提交事务。 
异常处理：  
1. 协调者故障：记录协调者的操作日志，重新拉起新的协调者后，根据操作日志向所有参与者询问状态，根据返回信息继续执行；
2. 参与者故障：由于协调者无法收到所有参与者的返回消息，导致协调者阻塞，设置协调者超时机制，指定时间内没有收到所有参与者执行写入成功的消息就全部回滚。   
存在的问题：
1. 协调者和参与者在Commit 阶段同时故障，导致新的协调者拉起来后不知道故障的参与者处于什么状态（是否 commit 成功），导致无法回滚
2. 单个参与者的超时或故障会阻塞所有的参与者，效率较低
## 4. 3PC 协议
在 2PC 协议的基础上，将 Prepare 阶段分为两个阶段：  
1. CanCommit 阶段：协调者向所有参与者发送 CanCommit 请求，各参与者根据自身是否能提交事务 commit 来响应请求。
2. PreCommit 阶段：如果所有参与者在 CanCommit 阶段都返回 Yes，则进入 PreCommit 阶段，各参与者执行写入但不提交事务。这一阶段参与者和协调者都引入超时机制（ 2PC 协议只对参与者有超时机制）
3. DoCommit 阶段：同 2PC Commit 阶段，唯一区别是参与者如果在超时时间内没有收到协调者发来的 Docommit 请求则自动提交 commit。   

相较于 2PC，3PC 解决了问题 2 降低了单个参与者阻塞对整体的影响，但问题 1 仍无法解决。    
3PC 协调者出现超时执行者自动 commit 还引入了新的问题：如果部分协调者出现了网络异常接收不到来自协调者的回滚请求，则会超时自动 commit，网络恢复后集群出现分区不能保证一致性
## 5. Paxos 协议

## 6. Raft 协议
[论文原文](https://raft.github.io/raft.pdf) 
[论文译文](https://docs.qq.com/doc/DY0VxSkVGWHFYSlZJ?_t=1609557593539)
#### 6.1 基础
##### 6.1.1 角色成员
- **Leader** Leader 通过选举产生，负责处理客户端请求，并向 Follower 同步请求对应的日志，任意时刻系统内最多只有一个 Leader
- **Follower** Follower 接收并持久化 Leader 同步的请求日志，在 Leader 触发下提交日志
- **Candidate** Follower 竞选 Leader 时的临时角色，选举结束后消失
##### 6.1.2 选举
选举过程：
1. 每位 Leader 都对应一个任期（term），任期内 Leader 会接收客户端请求，并将请求对应的指令日志同步到各 Follower，同时也会向所有 Follower 发送定时心跳
2. 当 Follower 在超时时间后仍未收到来自 Leader 的心跳消息，则会等待一段随机时间后发起 Leader 选举
3. Follower 将自己的角色转换为 Candidate，向其他 Follower 发送 RequestVote RPC 请求，请求中的 term 为当前 term + 1
4. Candidate 根据投票结果产生对应变化：
    - 赢得选举：Candidate 收到来自多数 Follower 发送的赞成票（term 相同）就赢得选举，成为新的 Leader 集群进入新的 term
    - 其他 Candidate 获胜：收到来自其他 Candidate 的 AppendEntries RPC 请求，如果请求中的 term 大于等于该 Candidate 当前的 term 则承认败选转回 Follower 状态，否则拒绝请求并保持 Candidate 状态
    - 选举超时：如果多个 Candidate 瓜分选票都拿不到多数赞成票，Candidate 会因为超时而进入新一轮选举，将 term + 1 后发送新的 RequestVote RPC 请求

每个 Candidate 的选举超时时间采用固定区间内的随机值，避免长时间选票瓜分导致选举迟迟不能结束
##### 6.1.3 日志同步
1. **同步日志**     
Leader 接收客户端请求后，会将请求对应的操作日志追加到日志中，然后向其他 Followers 并行发送 AppendEntries RPC 请求来同步日志，当日志都同步成功后 Leader 执行请求返回结果。  
如图，每条日志包含了请求的操作、被创建时的 term、在日志中的索引。   
如果 Leader 成功将日志同步到了大多数节点上，就认为执行日志中的操作是安全的可以被执行，被称为 committed，Leader 会记录最新 committed 的日志的索引，并将其放入后面的 AppendEntries RPC 请求中，Follower 在接收到后会执行被 committed 的日志对应的操作。    
<img src="https://qqadapt.qpic.cn/txdocpic/0/17ad6a18d9c67d1362166afa6c159920/0?w=1928&h=1236" width="500px"/>
2. **一致性检查和修复不一致**   
Raft 利用 index 和 term 标识唯一的一条日志，不同节点上同一条日志之前的所有日志保证完全相同。通过一致性检查发现并修复不同的情况：   
    1. Leader 上任后向 Follower 发送 AppendEntries RPC 请求，如果请求中的 index 和 Follower 的最新 index + 1 不一致 Follower 会拒绝请求。   
    2. Leader 收到拒绝后表明一致性检查失败，Leader 会尝试对失败的 Leader 同步 index - 1 的日志并不断重复，直到同步到 Leader 和 Follower 一致的日志请求成功，这次同步会将 Leader 从这条一致日志之后的所有日志一并同步给 Follower，并覆盖掉不同步的部分，从而使 Leader 和 Follower 完全一致。 
    3. 步骤 2 还存在其他优化手段来降低发现不一致和同步的耗时    

##### 6.1.4 Safety 额外限制
在前面的基础上还需要进一步的限制才能保证 Leader 在指定的 term 包含全部已提交的日志
1. **选举限制**     
**Candidate 在拉选票时，Follower 只会在这位 Candidate 拥有 Follower 处全部 committed 日志记录的情况下，才会投票给它**。当 Candidate 赢下大多数选票胜选时，既表明新的 Leader 拥有和绝大多数节点相同的 committed 日志记录，又因为 Leader committed 日志的前提是日志成功同步到大多数节点，因此可以认为新 Leader 拥有全部已经 committed 的日志记录。这样就避免了需要从 Follower 同步日志到新的 Leader，使日志的同步只存在于 Leader 到 Follower
2. **对以前 term 日志的 commit**    
对于前任 term 已同步但未 commit 的日志记录，新任 Leader 不能根据这条记录是否被同步到多数节点，来判断其是否可以被 commit。**每任 Leader 只能 commit 属于现任 term 的日志记录**。     
<img src="https://qqadapt.qpic.cn/txdocpic/0/0d10ed6c377a446005e21e9638f9a30b/0?w=1626&h=1724" width="500px"/>      
当出现如上图所示的 Leader 交替 Crash 和 Recover 情况，可能会前一位 Leader 同步到多数节点的日志还未 committed， 后一位 Leader 开始同步下一条日志时会覆盖掉前任 term 同步后未 committed 的日志，因此前任 term 日志同步的份数不能作为 commit 的依据。只能通过在 commit 本任 term 的日志记录时利用一致性修复来将前任 term 的日志同步和 commit 到其他节点    

##### 6.1.5 Follower 和 Candidate Crash

#### 6.2 
## Zab 协议
## Gossip 协议
## CRAQ 协议
[论文原文](https://pdos.csail.mit.edu/6.824/papers/craq.pdf)