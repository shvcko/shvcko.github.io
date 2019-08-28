---
layout: post
title: "Raft 算法介绍"
categories: blog
tags: ["Raft", "共识算法", "分布式一致性"]
---

Raft 是相比 Paxos 更易理解的一种一致性算法，它提供了和 Paxos 算法相同的功能和性能，但它的算法结构和 Paxos 不同。为了提升可理解性，Raft 将一致性算法分解成了几个关键模块，例如领导人选举、日志复制、安全性等。同时它通过实施一个更强的一致性来减少需要考虑的状态的数量。

Raft 在许多方面与现有的一致性算法类似（特别是 Oki 和 Liskov 的Viewstamped Replication [29,22]），但它有一些独特的地方：
* 强领导者：Raft 强化了领导者能力。比如，日志条目只能从领导者发送给其他的服务器。这种方式简化了对复制日志的管理并且使得 Raft 算法更加易于理解。
* Leader 选举：Raft 算法使用随机计时器来选举领导者。
* 成员关系变更：Raft 使用一种 joint consensus 机制来处理集群成员变更。

**Raft 基础**

Raft 允许系统半数以下节点失效。在任意时刻，每个服务器节点都处于这三种状态之一：Leader（领导者）、Follower（跟随者）或者 Candidate（候选人）。正常情况下，有且只有一个 Leader，其他全是 Follower。Candidate 只有在竞选情况下才会出现。

Raft 将时间划分为一个个任意长度的 term（以连续整数编号）。每个 term 都起始于一个 leader 选举，如果一个候选人赢得选举，然后他就在接下来的任期内充当领导人的职责，有些情况下，由于选票被瓜分，一个 term 会以没有领导人形式结束。term 在 Raft 中充当了逻辑时钟的作用，每个节点都保存了当前 term 的编号，一个随时间单调递增的数字，它可以用来检测过期的信息。比如：一个 Candidate 或一个 Leader 发现它的 term 已经过期，那么它们会立刻将自己将为 Follower 状态，如果一个服务器接收到了带有过期 term 编号的请求，那它将会拒绝这个请求。

Raft 使用 RPC 进行节点间通信。基础一致性算法里只要求两种类型的 RPCs：RequestVote RPCs 在 leader 选举期间发起， AppendEntries RPCs 在复制日志及发送心跳时发起（第七节介绍了第三种 RPC 用于传输 snapshot 快照）。

**Raft Leader 选举**

Raft 使用心跳机制来触发 leader 选举。正常情况下，Leader 会定期发送心跳信息至其他节点以维持它的身份。一旦一个 Follower 在一段时间内都没收到 leader 的心跳信息，我们称之为 election timeout，那它就会认为已经没有可用的 leader 节点，就会马上转变为 Candidate 开始发动选举（节点刚启动时，角色都是 Follower）。

一个候选人接收到同一个 term 的多数人的投票之后就赢得了选举，每个节点在同一个 term 里面只会以先到先得原则投票给一个候选人。多数投票机制确保了只有一个候选人会赢得选举，候选人在赢得选举后就发送心跳信息至所有其他节点以确立它的地位并防止产生新的选举。

在等待投票的过程中，候选人可能接收到其他节点用来声明要成为 leader 的 AppendEntries RPC 信息，如果这个信息的 term 编号大于等于当前节点 term 编号，那这个节点认可另外节点的合法领导人地位并转换自己身份为 Follower，如果 term 编号小于自己的编号，则它拒绝这个信息并继续进行选举。

为了防止选票尽可能的不被平均瓜分，Raft 使用了随机选举超时机制。election timeout 在一个固定范围（如：150-300ms）之间随机选取，这样使得很多情况下只有一个 server 会先 timeout，然后进行选取，当其他节点 timeout 时这个 server 的选举就已经完成并广播了心跳信息，这样就避免了冲突。

**日志复制**

Leader 接收到 Client 的命令请求后，首先将命令作为一个新条目添加到自己的日志中，接着并行发送 AppendEntries RPCs 请求到所有其他节点进行日志的复制。如果这个条目已被大多数节点接受，则 Leader 将这个条目应用到状态机中并返回结果给客户端。如果有的 Follower 节点故障或者失联了，则 Leader 会一直进行重试（即使已经返回响应给客户端），直到 Follower 节点同步完所有日志条目（为啥要甚至可以一直重试？因为 raft 机制保证了 leader 一直会发送同步信息给从节点，且如果大多数节点接受情况下即使换 leader 也能重复这个过程且从节点也必须接受这条日志以确保一致性，更多换 leader 情况下面介绍）。

一条日志由 term 编号、内容和一个位置索引值组成。

一旦一条日志被复制到多数集群节点，Leader 则认为这条日志是安全的，并可以应用到状态机，这样的日志称为 committed（意味着 committed，但并不需要什么操作）。Raft 保证 committed 日志是持久化的并最终会被状态机执行（在 leader 变更情况下 committed 的日志也可能是不安全的，需要更多限制，后续会介绍）。

在添加日志前，每个节点会进行一个简单的一致性检查，只有 leader 发送过来的日志条目（附带了新条目之前条目的信息）及前一条日志 term 编号和位置索引都对上才会同意接受。

正常情况下，这个一致性检查都会成功，在 leader 变更等情况下，主从节点间日志可能会不一致。Raft 使用 Leader 强制 Follower 同步自己的日志来处理非一致性，在这种情况下，从节点的冲突日志就有可能被覆盖，这也是为什么说 committed 日志仍不安全的原因，下面会说明如何增加一个限制来确保日志的安全。

Leader 维护了所有 Follower 的一个 nextIndex 值，用来标明 leader 将会发给 Follower 的下一条日志的位置，当冲突发生时，Follower 的 nextIndex 一致性检查就会失败，并回复拒绝信息给 leader，leader 这时就会对 nextIndex 减 1 并重新发送，如此重复直至一致性检查通过，此时 Follower 删除掉 nextIndex 后所有冲突的日志，并同步 Leader 数据。

**安全性**

上面讲了 Leader 选取和日志复制，但安全性方面仍有些不足。这里对 Raft 算法再做些完整性补充。

在任何基于领导人的一致性算法中，领导人都必须存储所有已经提交的日志条目。在某些一致性算法中，例如 Viewstamped Replication，某个节点即使是一开始并没有包含所有已经提交的日志条目，它也能被选为领导者。这些算法都包含一些额外的机制来识别丢失的日志条目并把他们传送给新的领导人，要么是在选举阶段要么在之后很快进行。不过这种方法会导致相当大的额外的机制和复杂性。Raft 使用了一种更加简单的方法保证所有之前的任期号中已经提交的日志条目在选举的时候都会出现在新的领导人中，不需要传送这些日志条目给领导人。这意味着日志条目的传送是单向的，只从领导人传给跟随者，并且领导人从不会覆盖自身本地日志中已经存在的条目。

Raft 在使用 RequestVote RPC 进行投票时，如果一个投票者发现自己的日志比候选人发送过来的日志信息更新，则它将拒绝这个候选人的投票。怎么判断日志是否更新呢？Raft 通过先比较日志的 term 编号，再比较日志的位置索引信息来确定。由于一个候选人必须和多数节点进行通信，而 committed 日志至少会存在于大多数集合中的一个节点中，这样就可以确保新选出来的 Leader 肯定包含了所有 committed 的信息。

另外，一个 Leader 节点在自己的 term 内可以及时知道哪条日志已经 committed 了，但切换领导人的情况下，新 Leader 并不能马上知道日志是否已经是 committed 了。由于这个差异就会导致已经 committed 的日志也会被覆盖（Raft 中举了一个例子 figure-8 说明了这个问题）。为了解决这个问题，Raft 不会通过是否已存在大多数节点上来确认 commit 一条之前 term 的日志，只有当前 Leader term 内才能采用这种方法。Leader 切换后，只有在当前 term 内 Leader 又 commit 了一条日志，才会顺带把之前日志也 commit 掉（通常新 Leader 会通过提交一个 no-op 日志来快速进行这个过程）。Raft 保留了之前 term 周期内日志的 term 编号，这样相比其他一致性算法虽然有些额外的复杂性但可使日志更易识别并减少了日志数据的发送。

Raft 的要求之一就是安全性不能依赖时间：整个系统不能因为某些事件运行的比预期快一点或者慢一点就产生了错误的结果。但是，系统可用性（系统及时响应客户的能力）不可避免的要依赖于时间。如果消息交换时间比服务器故障间隔时间长，候选人将没有足够长的时间来赢得选举；没有一个稳定的领导人，Raft 将无法工作。

领导人选举是 Raft 对时间要求最高的方面。Raft 只有在系统满足以下时间要求的情况下才能完成选举并维持一个稳定的领导者：

广播时间（broadcastTime） << 选举超时时间（electionTimeout） << 平均故障间隔时间（MTBF）

广播时间就是发送 RPC 并接受响应的时间，选举超时时间如之前所述，平均故障时间指集群中节点的故障发生平均间隔。

选举超时时间必须大于广播时间，这样候选人才能发送选举请求，并通过发送心跳来保持选举成功后的领导者地位。通过随机化选举超时时间的方法，这个不等式也使得选票瓜分的情况变得不可能。选举超时时间应该要比平均故障间隔时间小上几个数量级，这样整个系统才能稳定的运行。当领导人崩溃后，**整个系统会大约相当于选举超时的时间里不可用**；我们希望这种情况在整个系统的运行中很少出现。

**集群成员变化**

为了让配置修改机制能够安全，那么在转换的过程中不能够存在任何时间点使得两个领导人同时被选举成功在同一个任期里。不幸的是，任何服务器直接从旧的配置直接转换到新的配置的方案都是不安全的。一次性自动的转换所有服务器是不可能的，所以在转换期间整个集群存在划分成两个独立的大多数群体的可能性。

为了保证安全性，配置更改必须使用两阶段方法。目前有很多种两阶段的实现。例如，有些系统在第一阶段停掉旧的配置所以集群就不能处理客户端请求；然后在第二阶段在启用新的配置。在 Raft 中，集群先切换到一个过渡的配置，我们称之为 joint consensus；一旦 joint consensus 已经被提交了，那么系统就切换到新的配置上。joint consensus 是老配置和新配置的结合：
Raft 先立即进入一个中间状态，叫 joint-consensus；这个状态包含旧配置和新配置的所有机器；但是 decision-making，如 leader election 和 log commitment，需要同时满足旧配置的大多数和新配置的大多数（need majority of both old and new configurations for election and commitment）；Leader 通过提交 Cold+new 和 Cnew 两条日志条目来进行两阶段操作，实质上，将整个集群可能的 decison-making 依赖的 majority 状态空间分成了 5 个阶段，分别是：old 中大多数做决定  ->（old 中的大多数做决定） 或者 （old 和 new 中的大多数同时做决定） ->（old 和 new 中的大多数同时做决定）-> （old 和 new 中的大多数同时做决定） 或者 （ new 中的大多数做决定） -> new 中的大多数做决定。这种状态变迁的核心与关键在于：这种变迁是渐进的，每一个阶段都不会产生 decison-conflict，从而保证变更过程的平稳。如果不这样做，直接变更 a -> b，显然这种粗暴的做法会导致decison-conflict。）

**其他**

略

参考资料：  
[1] In Search of an Understandable Consensus Algorithm.  
[2] In Search of an Understandable Consensus Algorithm 中文版翻译（https://www.infoq.cn/article/raft-paper）  
[3] In Search of an Understandable Consensus Algorithm 中文版翻译（https://github.com/maemual/raft-zh_cn/blob/master/raft-zh_cn.md）  
[4] 分布式一致性算法Raft简介(上/下)（https://www.jianshu.com/p/ee7646c0f4cf）  
[5] Ongaro 和 Ousterhout 在 youtube 上的分享（https://link.jianshu.com/?t=http://youtu.be/YbZ3zDzDnrw）

