---
layout: post
title: "PacificA 论文阅读笔记"
categories: blog
tags: ["PacificA", "分布式一致性"]
---

PacificA，一个基于日志的分布式存储系统原型。PacificA 论文对其简单、实用且强一致性等特性做了详细介绍，为其他基于日志的分布式副本复制系统开发提供了良好的经验和建议。

在 PacificA 架构设计中，为了保证简单和模块化，应用了如下设计原则：
1. 分离副本组配置（replica-group configuration）管理和数据复制， Paxos 集群负责配置管理，主/从（primary/backup）机制用于数据复制
2. 故障检测和触发重配置去中心化，并使监控流量可以遵循数据复制模式运行
3. 提炼了一个通用的抽象模型，可以方便的验证正确性，且允许实例化不同类型的实现

下面是对框架具体实现的介绍。

**PacificA 复制框架**

PacificA 维护一个大数据集，每份数据都复制在一组服务器上，一组复制节点称为一个副本组（replica-group）。一个服务器可以为多个副本组提供服务。数据复制协议遵循主/从范式（primary/backup paradigm）。副本组中的一个副本被定为主副本（primary），其他为从副本（secondaries）。副本组的配置（replica-group  configuration），用于说明谁是主副本谁是从副本，在副本故障或新加入时产生变更，并使用版本号记录这些变更。

PacificA 只关注*强一致的副本复制协议*，因为它是最自然的一致性模型，并且在有副本和没有副本情况下它可以保持语义一致（如线性一致性）。

**主/从数据复制**

PacificA 采用 primary/backup 范式进行数据复制。将客户端请求分为两种：query 请求和 update 请求，query 请求不修改数据，update 请求则会对数据进行修改。*在 primary/backup 范式中，所有请求都发往主节点*。主节点处理所有请求，其他所有从节点则只处理 update 请求。 primary/backup 范式在实践中很有吸引力，它简单，并且非常接近于没有副本只有单个主节点提供访问的场景。

只要所有的节点以相同的顺序执行相同的命令集合，结果就能保证强一致性。因此，主节点可以为每个 *update 请求*分配一个持续的单调递增编号（serial number），并要求从节点都按照编号的顺序执行。为清楚起见，PacificA 为每个副本构建了一个请求的预备队列（prepared list），以及这个队列上的一个提交点（committed point）。这个队列遵循主节点分配的请求编号顺序，并且确保编号连续（也即，编号为 sn 的请求只有在编号为 sn-1 的请求已插入的情况下才会插入队列）。committed point 之前的队列前缀也称为提交队列（committed list）。

应用程序的状态就是初始状态下顺序、递增执行 committed list 中所有请求的结果，*在 committed list 中的请求我们保证不会丢失*（除非发生无法容忍的故障，比如所有副本节点都发生了永久故障）。prepared list 中未提交的后缀用于确保重配置的过程中已在 committed list 中的请求能够保证保留。

正常情况下 query 和 update 协议。在正常情况下（也就是，没有发生重配置）数据的复制协议非常直接简单。对于 query 请求，当一个主节点接收到 query 请求时，它只要在应用程序状态（如上所述）执行查询并直接返回请求响应即可。对于 update 请求，当主节点 p 收到更新请求后，先给请求分配*下一个*可用的 sn 编号，然后将请求、当前的配置版本、sn 编号一起作为一个 prepare 消息发送给所有从节点。一个副本 r 收到 prepare 消息后，按照 sn 编号顺序将其插入 prepared list 列表中，此时，即认为该请求在 r 副本中已prepared。r 随即发送一个 prepared 确认信息回给主节点。当主节点收到*所有*的副本 prepared 确认回复信息后，该请求就被认为是 committed。 此时，主数据库将其提交点（committed point）向前移动到最高点，以便提交到此时为止的所有请求。然后主节点 p 发送确认信息给客户端，表明请求已成功完成。对于每个 prepare 消息，主节点可以进一步带上 committed point 的 sn 编号，用于通知从节点已提交的请求信息，从节点就可以相应的前移它的 committed point。

因为主节点只有在收到所有从节点的回复信息后才添加一个请求到它的 committed list（当向前移动 committed point 时），因此主节点的 committed list 始终是从节点（任何一个）的 prepared list 前缀，同样，因为从节点添加请求至 committed list 只有在主节点已完成之后才会进行，所以，从节点的 committed list 肯定是主节点的 committed list 前缀。因此，这个简单数据复制协议维持了一下提交不变性：

提交不变性：设 p 为主副本节点 q 为当前配置中的任一从副本节点，则有 committedq ⊆ committedp ⊆ preparedq。  
Commit Invariant: Let p be the primary and q be any replica in the current configuration, committedq ⊆ committedp ⊆ preparedq holds.

**配置管理**

在 PacificA 的设计中，一个全局配置管理器（global configuration manager）管理所有副本组的配置信息。对于每个副本组，配置管理器维护着它的当前配置和版本。

一个服务器会发起重配置操作，当他怀疑某个副本失效时（通过故障检测机制），重配置后失效的副本节点会从配置中移除。一个节点也可能提议增加一个副本节点到配置中。无论哪种情况，节点都需要发送提议的新的配置信息及当前配置信息版本到配置管理器，当且仅当发送的版本信息和配置管理器当前的版本信息互相匹配时，这个请求才会受理，在这种情况下，提议的新配置被接受并更新版本号至下一个更高版本。其他情况下，请求被拒绝。

在出现网络分区导致主节点与从节点连接断开时，有可能会出现冲突的重新配置请求：主服务器可能要删除某些从服务器，而某些从服务器尝试删除主服务器。由于所有这些请求都基于当前配置，因此配置管理器接受的第一个请求获胜。所有其他冲突的请求都会被拒绝，因为配置管理器已经进入具有更高版本的新配置。

任何故障检测机制只要确保以下主副本节点不变性就可用于触发删除当前副本。

主副本节点不变性：在任何时候，一个节点 p 只有在配置管理器维护的当前配置中 p 是主节点时它才能认为自己是主节点。因此，在任何时候，一个副本组中最多只有一个节点会认为自己是主副本节点。  
Primary Invariant: At any time, a server p considers itself a primary only if the configuration manager has p as the primary in the current configuration it maintains. Hence, at any time, there exists at most one server that considers itself a primary for the replica group.

**租约和故障检测**

即使配置管理器中维护了正确的配置，Primary Invariant 也不一定能成立，因为不同的节点可能有不同的本地视图，它们不一定完全同步。在现实中，我们必须要避免新的主节点和旧的主节点同时提供服务处理请求——因为旧的节点可能没有感知到新的节点已经产生并且自身已在配置中被被移除。

PacificA 的解决的方案是使用租约机制。主节点通过周期性的给每个从节点发送信标并等待响应来获得每个从节点的租约，在收到最后的信标响应一段指定租约期（lease period）后，主节点会认为其租约已过期。当任何一个来自从节点的租约过期时，主节点*不再*认为其仍是主节点，并停止处理 query 及 update 请求。然后联系配置管理器将过期的副本从当前配置中移除。

关于从节点这边，只要信标的发送者仍是当前配置中的主节点，它就是确认这个信标。如果自从上次收到主节点的信标后经过一段指定的宽限期（grace period）后仍未再次收到信标，则从节点认为主节点租约已过期，它将联系配置管理器移除主节点并将其自己设为主节点。

假设零时钟漂移，只要宽限期与租约期相同或更长，主节点租约就会保证在从节点租约到期之前到期。而一个从节点当且仅当在旧的主节点租约到期后才会发起提议变更配置并使自己成为主节点，因此可以保证在新的主节点确立之前旧的主节点已经辞职。因此 Primary Invariant 成立。

GFS、BigTable/Chubby 中也使用了类似的租约机制作为故障检测机制。不过有一点关键的区别是，在这些系统中，租约从一个中心实体中获取，而在 PacificA 的例子中，故障检测的监控流量和两个节点间必要的数据处理流量合二为一：因为当处理 update 请求时主节点本身需要和所有从节点通信，信标及其确认信息也需要在主节点和从节点之间通信。通过这种方式，故障检测就已经可以准确的获取到通信通道的状态。并且，当通信信道繁忙时，数据处理消息本身就可以用作信标和确认信息。仅当通信信道空闲时才发送实际信标和确认，从而最小化故障检测的开销。此外，这种去中心化的实现消除了对集中式实体的负载和依赖性。集中式实体的负载是很重要的，因为信标和确认消息需要在集中式实体和系统中的每个服务器之间定期交换，为了便于快速检测故障，信标的发送间隔又必须相当小。在集中式解决方案中，集中式实体的不可用（例如，由于网络分区）可能会导致整个系统不可用，因为从中央实体获取的租约到期后所有主节点都必须辞职。

**Reconfiguration、Reconciliation 和 Recovery**

复制协议的复杂性在于它如何处理重配置，PacificA 将重配置分为三种类型：移除从节点、移除主节点和新增从节点。

移除从节点在主节点怀疑一个或多个从节点失效时触发，主节点提议一个不包含这些节点的新配置，如果配置管理器接受提议则主节点可以和剩余的从节点继续。

主节点变更。当一个从节点怀疑主节点失效时触发移除主节点操作（如上租约机制），从节点提议一个新配置让其自身成为主节点并去除旧的主节点。如果提议通过，则从节点 p 变为主节点，但并不立即开始执行请求操作，首先需要执行一个 reconciliation 流程。在 reconciliation 流程中，新的主节点 p 发送其 prepared list 列表中所有未 committed 的请求的 prepare 消息，以在新的配置下提交它们。设 sn 为 p 在 reconciliation 的 committed point，p 接着发通知指示所有的从节点将其 prepared list 截断并只保留请求数据至 sn 这个位置。如果 reconciliation 期间新配置中的某个副本又失效了，有可能会触发另一个重配置流程。

因为一个从节点的 prepared list 始终包含所有 committed 的请求，通过促使所有在 prepared list 中的请求 committed， reconciliation 保证了以下重配置不变性：

重配置不变性：如果一个新的主节点 p 在 t 时间点完成了 reconciliation 流程，则副本组中任何副本维护的任何 committed list 在 t 时间点之前肯定是主节点 p 的 committed list 前缀。Commit Invariant 在新配置中成立。  
Reconfiguration Invariant: If a new primary p completes reconciliation at time t, any committed list that is maintained by any replica in the replica group at any time before t is guaranteed to be a prefix of committedp at time t. Commit Invariant holds in the new configuration.

重配置不变性（Reconfiguration Invariant）跨配置扩展了提交不变性（Commit Invariant），并且确保分配给任何已提交请求的 sn 编号将被保留，即使在主节点有变更的情况下。不过，prepared list 尚未提交的部分则不一定能够保留，新的主节点可以将一个编号赋予一个新的请求，只要这个编号尚未赋给已提交的请求。从节点总是选择来自具有最高配置版本的主节点的分配编号。

::【备注】按 PacificA 协议，会导致数据重复：新主节点 np 的 prepared list 超过旧主节点 op 的 committed list 部分，在 reconciliation 流程后会被 np 提交，成为 committed list 一部分，而由于之前 op 并未将这部分请求返回成功 ack 给客户端，客户端理应会重试，此时会导致这部分数据重复。::

增加新的从节点。新的从节点加入时，Commit Invariant 条件必须维持，因此需要新的从节点在加入副本组之前必须先有合适的 prepared list（实际上，加入服务器会复制当前已提交的状态，以及尚未提交的已准备列表的部分）。

一个简单的方法是主节点推迟所有处理请求，直至新的从节点从现有节点中完成 prepared list 拷贝，不过这在现实中太具破坏性。相应的，我们可以新增一个候选（Candidate）状态，从副本新加入时，先进入 Candidate 状态，主节点可以继续正常处理请求，并且在收到 update 请求时，也发给 Candidate 节点。一旦主节点接收不到 Candidate 节点的确认消息，主节点就可以终止它的候选人资格。

Candidate 节点 c 在处理主节点发过来的 update 请求同时，也同步去拉取缺失的 prepared list 部分（可以从任何节点），最终追上了的时候，c 就可以请求主节点其加入配置文件中，只要此时 c 的候选人资格没有被终止，主节点就会联系配置管理器将其加入配置。一经批准，主节点就会仍同 c 作为从节点。

新副本节点加入时状态的迁移恢复是一个代价很高的操作，并且对于一个从未在副本组中的节点来说全量迁移是不可避免的。不过对于其他的一些场景，如网络分区，或其他一些临时故障导致副本移除出副本组又重新加入的情况下，可以采用 catch-up 恢复策略，只需恢复不在副本组期间遗失的内容。基于重配置不变性条件，旧从节点的 committed list 可以肯定保证是当前有效 committed list 和 prepared list 的前缀（旧从节点的 prepared list 则不一定能保证），因此，新恢复的节点只需要从 committed list 中的 committed point 进行追赶恢复即可。

**正确性**

略。

**实现**

略。

**分布式日志复制存储系统架构**

PacificA 是较为经典的分布式存储系统设计。整个系统由 application log + In-Memory structure + checkpoint + On-Disk Image + New On-Disk Image 组成。并分别介绍了 1、逻辑复制（Logical Replication）——每个节点都是独立副本机制，采用 application log + prepared list 合二为一，先写 application log 再写内存结构，一旦提交 checkpoint 就可删除对应 application log 日志，记录 checkpoint 和 on-disk image 的 sn 编号，以方便 catch-up 恢复；1.1、逻辑复制的变种（logical-V）——只在 primary 执行上述流程，从节点只执行将主节点发送的 update 请求写日志，不写内存，并从主节点拉取 checkpoint。这可以减少从节点 cpu 和内存，但增加了带宽和恢复时间；2、分层复制（Layered Replication），存储和计算分离（最近的分布式存储系统都如此设计，如 BigTable on GFS），计算流程和上述类似，存储层采用单独分布式存储系统如 GFS。3、Log Merging，略。

参考资料：  
[1] PacificA: Replication in Log-Based Distributed Storage Systems.

