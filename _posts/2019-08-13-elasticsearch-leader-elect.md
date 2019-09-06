---
layout: post
title: "Elasticsearch 选主流程"
categories: blog
tags: ["Elasticsearch", "Bully 算法", "es 选主流程", "分布式一致性", "源码解析"]
---

Elasticsearch 使用了一个类似 Bully 的算法进行 Leader 选举。Bully 算法在使用上有一些预设条件，比如系统需要是同步的（the system is synchronous）、有节点/进程故障监测机制（there is a failure detector which detects failed processes）、每个节点/进程都知晓自身及其他所有节点/进程的 id 和通信地址（each process knows its own process id and address, and that of every other process）等。Bully 算法总是选出进程中 id 最大的作为 Leader，其逻辑相较于其他分布式一致性算法而言较为简单。Elasticsearch  中，Discovery 模块用于节点发现和 Leader 选举，其默认内置的 ZenDiscovery 实现了 Bully 相关的内容。在具体实现上，又分为节点发现、选举 Leader、发布状态、失效监测等几个部分，下面按 es 启动时的实际执行流程介绍一下各部分内容。

## 节点发现（ZenPing）
一个节点启动时，会在其启动函数 start() 中进行初始化 Discovery 模块、注册集群状态发布函数和执行加入集群（Join）等操作，选主流程主要在加入集群 startInitialJoin() 方法中。相关代码如下：
```java
public Node start() throws NodeValidationException {
    ...
    Discovery discovery = injector.getInstance(Discovery.class);
    clusterService.getMasterService().setClusterStatePublisher(discovery::publish);
    ....
    discovery.start(); // start before cluster service so that it can set initial state on ClusterApplierService
    ...
    discovery.startInitialJoin();
}
```

startInitialJoin() 中会启动一个新的线程去执行集群 Join 工作
```java
@Override
public void startInitialJoin() {
    // start the join thread from a cluster state update. See {@link JoinThreadControl} for details.
    synchronized (stateMutex) {
        // do the join on a different thread, the caller of this method waits for 30s anyhow till it is discovered
        joinThreadControl.startNewThreadIfNotRunning();
    }
}
```

es 使用一个 JoinThreadControl 结构对集群 Join 操作及集群状态变更进行控制，以确保有且只有一个线程在执行 Join 及状态变更相关操作，并更进一步保证 Join 及状态变更等操作是同步的（参照上面 Bully 算法要求）。在 JoinThreadControl 的 startNewThreadIfNotRunning() 中，对上述条件进行了判断之后，通过 threadPool.generic().execute(new Runnable() {}) 启动新线程执行：
```java
/** starts a new joining thread if there is no currently active one and join thread controlling is started */
public void startNewThreadIfNotRunning() {
    assert Thread.holdsLock(stateMutex);
    if (joinThreadActive()) {
        return;
    }
    threadPool.generic().execute(new Runnable() {
        @Override
        public void run() {
            Thread currentThread = Thread.currentThread();
            if (!currentJoinThread.compareAndSet(null, currentThread)) {
                return;
            }
            while (running.get() && joinThreadActive(currentThread)) {
                try {
                    innerJoinCluster();
                    return;
                } catch (Exception e) {
                    logger.error("unexpected error while joining cluster, trying again", e);
                    // Because we catch any exception here, we want to know in
                    // tests if an uncaught exception got to this point and the test infra uncaught exception
                    // leak detection can catch this. In practise no uncaught exception should leak
                    assert ExceptionsHelper.reThrowIfNotNull(e);
                }
            }
            // cleaning the current thread from currentJoinThread is done by explicit calls.
        }
    });
}
```

innerJoinCluster() 是 Join 操作的主函数，它的主要流程分以下几步：1、进行节点发现及查找主节点，相关代码主要在 findMaster() 中；2、根据主节点查找结果，判断当前节点是否是主节点候选人。如果是，则进行“成为主节点”（登基？）操作，相关代码主要在 waitToBeElectedAsMaster()，如主节点候选人另有其人，则进行“加入主节点集群”（打不过它，就加入它？）操作，相关代码主要在 joinElectedMaster()；3、以上任何步骤出错，则标记停止当前选举线程，并另起一个新的线程重新执行选举过程；4、Join 操作完成后，启动失效监测机制，定时检测主节点或普通节点是否出现故障。
```java
/**
 * the main function of a join thread. This function is guaranteed to join the cluster
 * or spawn a new join thread upon failure to do so.
 */
private void innerJoinCluster() {
    ...
    while (masterNode == null && joinThreadControl.joinThreadActive(currentThread)) {
        masterNode = findMaster();
    }
    ...
    if (transportService.getLocalNode().equals(masterNode)) {
        ...
        nodeJoinController.waitToBeElectedAsMaster(requiredJoins, masterElectionWaitForJoinsTimeout,
                new NodeJoinController.ElectionCallback() {
                    ...
                }
        );
    } else {
        ...
        // send join request
        final boolean success = joinElectedMaster(masterNode);
        ...
    }
}
```

首先，看一下 findMaster() 的逻辑。es 中，寻找主节点的第一步，是需要先做节点发现，在 findMaster() 中，首先使用 ZenPing 寻找和发现所有集群节点：
```java
private DiscoveryNode findMaster() {
    logger.trace("starting to ping");
    List<ZenPing.PingResponse> fullPingResponses = pingAndWait(pingTimeout).toList();
    if (fullPingResponses == null) {
        logger.trace("No full ping responses");
        return null;
    }
    ...
}
```

在 pingAndWait() 中，调用 ZenPing 的 ping() 接口，进入具体的实现 UnicastZenPing 的 ping 方法：
```java
protected void ping(final Consumer<PingCollection> resultsConsumer,
                    final TimeValue scheduleDuration,
                    final TimeValue requestDuration) {
    final List<TransportAddress> seedAddresses = new ArrayList<>();
    seedAddresses.addAll(hostsProvider.buildDynamicHosts(createHostsResolver()));
    final DiscoveryNodes nodes = contextProvider.clusterState().nodes();
    // add all possible master nodes that were active in the last known cluster configuration
    for (ObjectCursor<DiscoveryNode> masterNode : nodes.getMasterNodes().values()) {
        seedAddresses.add(masterNode.value.getAddress());
    }
    ...
    final PingingRound pingingRound = new PingingRound(pingingRoundIdGenerator.incrementAndGet(), seedAddresses, resultsConsumer,
            nodes.getLocalNode(), connectionProfile);
    activePingingRounds.put(pingingRound.id(), pingingRound);
    final AbstractRunnable pingSender = new AbstractRunnable() {
        ...
        @Override
        protected void doRun() throws Exception {
            sendPings(requestDuration, pingingRound);
        }
    };
    threadPool.generic().execute(pingSender);
    threadPool.schedule(TimeValue.timeValueMillis(scheduleDuration.millis() / 3), ThreadPool.Names.GENERIC, pingSender);
    threadPool.schedule(TimeValue.timeValueMillis(scheduleDuration.millis() / 3 * 2), ThreadPool.Names.GENERIC, pingSender);
    threadPool.schedule(scheduleDuration, ThreadPool.Names.GENERIC, new AbstractRunnable() {
        @Override
        protected void doRun() throws Exception {
            finishPingingRound(pingingRound);
        }
        ...
    });
}
```

在 UnicastZenPing 的 ping() 方法中，首先构建节点发现种子地址，将用户配置的 unicast.hosts 地址及当前节点可能知道的所有 master 地址都加入种子地址列表，其中 createHostsResolver() 用于对用户配置的 hosts 地址进行解析。并随后，构建每一轮 ping 所需要的 PingingRound 结构及 ping 发送者 pingSender，在 pingSender 的 doRun() 中执行发送 ping 请求。相关的结构构造好之后，ping 使用 threadPool 进行真正执行操作。一次 ping 执行总共有 3 轮（可参考上面 threadPool.schedule(…) 部分），第一轮马上执行，第二轮在 ping_timeout（discovery.zen.ping_timeout配置，默认 3s）1/3 处执行，第三轮在 ping_timeout 2/3 处执行，最后执行 finishPingingRound，在 ping_timeout 结束时。finishPingingRound 执行后，即使有还未完成的 ping 请求也将被停止（调用了 CompletableFuture 的 complete()），以便返回最终结果。

具体执行 ping 时，sendPings 构造 UnicastPingRequest 请求，并获取当前节点已获取到的和已知道的所有节点地址，以及用户种子地址列表，批量给所有这些地址发送 ping 消息。ping 结果的处理在 getPingResponseHandler 中的 handleResponse()，它会通过 pingingRound::addPingResponseToCollection 将所有 ping 响应加入到一个 Map<DiscoveryNode, PingResponse> 结构 pingCollection 中（由于是 Map 结构，相同节点的最新 response 会覆盖旧 response），以作为最后 ZenPing 结果返回给 findMaster()。这部分代码分别在 sendPings()，sendPingRequestToNode() 和 getPingResponseHandler() 中，大体如下所示：
```java
protected void sendPings(final TimeValue timeout, final PingingRound pingingRound) {
    ...
    final UnicastPingRequest pingRequest = new UnicastPingRequest(pingingRound.id(), timeout, createPingResponse(lastState));

    List<TransportAddress> temporalAddresses = temporalResponses.stream().map(pingResponse -> {
        ...
        return pingResponse.node().getAddress();
    }).collect(Collectors.toList());

    final Stream<TransportAddress> uniqueAddresses = Stream.concat(pingingRound.getSeedAddresses().stream(),
        temporalAddresses.stream()).distinct();
    ....
    nodesToPing.forEach(node -> sendPingRequestToNode(node, timeout, pingingRound, pingRequest));
}

private void sendPingRequestToNode(final DiscoveryNode node, TimeValue timeout, final PingingRound pingingRound,
                                   final UnicastPingRequest pingRequest) {
    submitToExecutor(new AbstractRunnable() {
        @Override
        protected void doRun() throws Exception {
            ...
            transportService.sendRequest(connection, ACTION_NAME, pingRequest,
                TransportRequestOptions.builder().withTimeout((long) (timeout.millis() * 1.25)).build(),
                getPingResponseHandler(pingingRound, node));
        }
        ...
    });
}

protected TransportResponseHandler<UnicastPingResponse> getPingResponseHandler(final PingingRound pingingRound,
                                                                               final DiscoveryNode node) {
    return new TransportResponseHandler<UnicastPingResponse>() {
        ...
        @Override
        public void handleResponse(UnicastPingResponse response) {
            ...
            } else {
                Stream.of(response.pingResponses).forEach(pingingRound::addPingResponseToCollection);
            }
        }
        ...
    };
}

public void addPingResponseToCollection(PingResponse pingResponse) {
    if (localNode.equals(pingResponse.node()) == false) {
        pingCollection.addPing(pingResponse);
    }
}
```

以上是 ping 发送端的逻辑，下面再看下接收端的处理。UnicastZenPing 注册了 UnicastPingRequestHandler 用于对 internal:discovery_zen_unicast 消息进行处理。在 UnicastPingRequestHandler 的 messageReceived() 中对接收到的 ping 进行了处理：
```java
private UnicastPingResponse handlePingRequest(final UnicastPingRequest request) {
    ...
    temporalResponses.add(request.pingResponse);
    // add to any ongoing pinging
    activePingingRounds.values().forEach(p -> p.addPingResponseToCollection(request.pingResponse));
    threadPool.schedule(TimeValue.timeValueMillis(request.timeout.millis() * 2), ThreadPool.Names.SAME,
        () -> temporalResponses.remove(request.pingResponse));

    List<PingResponse> pingResponses = CollectionUtils.iterableAsArrayList(temporalResponses);
    pingResponses.add(createPingResponse(contextProvider.clusterState()));

    return new UnicastPingResponse(request.id, pingResponses.toArray(new PingResponse[pingResponses.size()]));
}

class UnicastPingRequestHandler implements TransportRequestHandler<UnicastPingRequest> {
    @Override
    public void messageReceived(UnicastPingRequest request, TransportChannel channel) throws Exception {
        ...
        if (request.pingResponse.clusterName().equals(clusterName)) {
            channel.sendResponse(handlePingRequest(request));
        } else {
        ...
    }

}
```

消息处理的实际逻辑在 handlePingRequest() 中。首先，添加当前接收到的请求到 temporalResponses，temporalResponses 收集了所有给当前节点发送过 ping 请求的其他节点的 ping-response 结构信息，用于回复当前节点接收到的 ping 请求（由此可见，es 集群中只要一个节点有了其他节点信息，就能把这些信息带到向它发请求的所有节点中去，也就是 es 各节点能快速的互相彼此发现）。然后，将这次 ping-response 也加到还存活的 ping 活动的结果中去（自己送上门的 ping-response 和刻意发送 ping 请求去获取 response 结果是一致的，还能起到类似多加一轮 ping 的效果，这算是 es 的一个小技巧，有效的利用了发送 ping 时的请求）。接着，再起一个定时器定期清除 temporalResponses 中的 ping-response 信息，以保证响应内容的时效性。最后，就是构造返回结果信息，将现有接收到的所有其他节点 response 信息加上自身的 response 信息（自增 id，集群名，当前节点，主节点，集群版本）构造为一个 UnicastPingResponse 返回。

## 选举 Leader
回到 findMaster() 方法中，当 ZenPing 结束了 3 轮 ping 操作，并取回相关的结果信息后（fullPingResponses），就可以进行 Leader 节点的选举了。
```java
private DiscoveryNode findMaster() {
    logger.trace("starting to ping");
    List<ZenPing.PingResponse> fullPingResponses = pingAndWait(pingTimeout).toList();
    if (fullPingResponses == null) {
        logger.trace("No full ping responses");
        return null;
    }
    ...

    final DiscoveryNode localNode = transportService.getLocalNode();

    // add our selves
    assert fullPingResponses.stream().map(ZenPing.PingResponse::node)
        .filter(n -> n.equals(localNode)).findAny().isPresent() == false;

    fullPingResponses.add(new ZenPing.PingResponse(localNode, null, this.clusterState()));

    // filter responses
    final List<ZenPing.PingResponse> pingResponses = filterPingResponses(fullPingResponses, masterElectionIgnoreNonMasters, logger);

    List<DiscoveryNode> activeMasters = new ArrayList<>();
    for (ZenPing.PingResponse pingResponse : pingResponses) {
        // We can't include the local node in pingMasters list, otherwise we may up electing ourselves without
        // any check / verifications from other nodes in ZenDiscover#innerJoinCluster()
        if (pingResponse.master() != null && !localNode.equals(pingResponse.master())) {
            activeMasters.add(pingResponse.master());
        }
    }

    // nodes discovered during pinging
    List<ElectMasterService.MasterCandidate> masterCandidates = new ArrayList<>();
    for (ZenPing.PingResponse pingResponse : pingResponses) {
        if (pingResponse.node().isMasterNode()) {
            masterCandidates.add(new ElectMasterService.MasterCandidate(pingResponse.node(), pingResponse.getClusterStateVersion()));
        }
    }

    if (activeMasters.isEmpty()) {
        if (electMaster.hasEnoughCandidates(masterCandidates)) {
            final ElectMasterService.MasterCandidate winner = electMaster.electMaster(masterCandidates);
            logger.trace("candidate {} won election", winner);
            return winner.getNode();
        } else {
            // if we don't have enough master nodes, we bail, because there are not enough master to elect from
            logger.warn("not enough master nodes discovered during pinging (found [{}], but needed [{}]), pinging again",
                        masterCandidates, electMaster.minimumMasterNodes());
            return null;
        }
    } else {
        assert !activeMasters.contains(localNode) : "local node should never be elected as master when other nodes indicate an active master";
        // lets tie break between discovered nodes
        return electMaster.tieBreakActiveMasters(activeMasters);
    }
}
```

首先将本地节点也加入 fullPingResponses ，随后的 filterPingResponses() 方法可以对 fullPingResponses 做过滤，视用户的配置情况是否过滤掉非 master 角色节点的 ping 结果。然后，循环过滤结果对是否有现役的 Master 节点进行查找，需要注意的是在这里我们要避免将本地节点视为 activeMaster，以防止未经其他节点验证就选举出本地节点为 Leader，现役 Master 节点列表存放于 activeMasters。接着，查找所有符合条件的 Master 节点候选者，加入 masterCandidates 列表中。

如果查找出的 activeMasters 为空，也即目前还没有 Leader 节点，则先判断是否有满足要求数量的 Master 候选者。如果没有，记录警告信息并返回 null；如果数量满足，则进行 Master 选举。
```java
/**
 * Elects a new master out of the possible nodes, returning it. Returns {@code null}
 * if no master has been elected.
 */
public MasterCandidate electMaster(Collection<MasterCandidate> candidates) {
    assert hasEnoughCandidates(candidates);
    List<MasterCandidate> sortedCandidates = new ArrayList<>(candidates);
    sortedCandidates.sort(MasterCandidate::compare);
    return sortedCandidates.get(0);
}
```

选举的机制也很简单，就是对候选者列表进行一个排序，然后挑选排序后列表的第一个值。具体的排序规则在 MasterCandidate::compare 中实现，compare 会首先比较 es 集群状态版本，集群状态（clusterState）版本高的节点意味着拥有更新的状态，更适合选举为主节点，因此将会排在前面；如果版本一致，则进而比较节点角色，master 角色的节点优先；如果都是 master 角色节点，则再进而比较节点 id 值，由于节点 id 是一个全局唯一字符串，故以字符串字典序进行排序并将结果进行返回。
```java
/**
 * compares two candidates to indicate which the a better master.
 * A higher cluster state version is better
 *
 * @return -1 if c1 is a batter candidate, 1 if c2.
 */
public static int compare(MasterCandidate c1, MasterCandidate c2) {
    // we explicitly swap c1 and c2 here. the code expects "better" is lower in a sorted
    // list, so if c2 has a higher cluster state version, it needs to come first.
    int ret = Long.compare(c2.clusterStateVersion, c1.clusterStateVersion);
    if (ret == 0) {
        ret = compareNodes(c1.getNode(), c2.getNode());
    }
    return ret;
}

/** master nodes go before other nodes, with a secondary sort by id **/
 private static int compareNodes(DiscoveryNode o1, DiscoveryNode o2) {
    if (o1.isMasterNode() && !o2.isMasterNode()) {
        return -1;
    }
    if (!o1.isMasterNode() && o2.isMasterNode()) {
        return 1;
    }
    return o1.getId().compareTo(o2.getId());
}
```

如果查找出的 activeMasters 不为空，则从现有的 activeMasters 中挑选 Leader 节点。首先也是对 activeMasters 进行排序，排序的方法和上述 compareNodes 相同（调用的同一个函数），然后选择 id 最小的节点。
```java
/** selects the best active master to join, where multiple are discovered */
public DiscoveryNode tieBreakActiveMasters(Collection<DiscoveryNode> activeMasters) {
    return activeMasters.stream().min(ElectMasterService::compareNodes).get();
}
```

## 成为 Leader
确定 Master 节点后，需要先判断自己是否被选中。如果自己被选为 Master 节点，则进入 waitToBeElectedAsMaster “成为主节点”流程，等待足够多的其他节点发送 Join 请求，并最终发布状态更新宣示自己成为主节点。
```java
public void waitToBeElectedAsMaster(int requiredMasterJoins, TimeValue timeValue, final ElectionCallback callback) {
    final CountDownLatch done = new CountDownLatch(1);
    ...

    try {
        // check what we have so far..
        // capture the context we add the callback to make sure we fail our own
        synchronized (this) {
            assert electionContext != null : "waitToBeElectedAsMaster is called we are not accumulating joins";
            myElectionContext = electionContext;
            electionContext.onAttemptToBeElected(requiredMasterJoins, wrapperCallback);
            checkPendingJoinsAndElectIfNeeded();
        }

        try {
            if (done.await(timeValue.millis(), TimeUnit.MILLISECONDS)) {
                // callback handles everything
                return;
            }
        } catch (InterruptedException e) {
        ...
        failContextIfNeeded(myElectionContext, "timed out waiting to be elected");
    ...
}
```

waitToBeElectedAsMaster() 方法做了一些初始化（electionContext.onAttemptToBeElected()）以及进入后首次判断是否已有足够的 Join 请求外（如果以满足则直接走成为 Leader 流程），其余时间都只是在等待，等待采用了 CountDownLatch 的 await() 实现。当然等待也有超时，时间为 discovery.zen.master_election.wait_for_joins_timeout，默认为 30s。如果超时则停止当前选举，重头开始进行新的选举流程。等待的过程中，节点还在进行接收 Join 操作，不过这部分流程异步进行，在 internal:discovery_zen_join 的处理类 JoinRequestRequestHandler 中。

JoinRequestRequestHandler 重写了 messageReceived 实现，其内部调用 onJoin()，onJoin 又调用 handleJoinRequest：
```java
private class JoinRequestRequestHandler implements TransportRequestHandler<JoinRequest> {
    @Override
    public void messageReceived(final JoinRequest request, final TransportChannel channel) throws Exception {
        listener.onJoin(request.node, new JoinCallback() {...}
}

private class MembershipListener implements MembershipAction.MembershipListener {
    @Override
    public void onJoin(DiscoveryNode node, MembershipAction.JoinCallback callback) {
        handleJoinRequest(node, ZenDiscovery.this.clusterState(), callback);
    }
    ...
}
```


handleJoinRequest() 中，首先通过发送一个 internal:discovery_zen_join/validate 请求， 对 Join 请求合法性进行验证，合法性验证规则主要包括节点版本、索引合法性验证，以及用户自定义 Discovery 插件可能加入的其他验证规则等。通过验证后调用 nodeJoinController.handleJoinRequest(node, callback) 进行最终处理。
```java
void handleJoinRequest(final DiscoveryNode node, final ClusterState state, final MembershipAction.JoinCallback callback) {
    ...
        // we do this in a couple of places including the cluster update thread. This one here is really just best effort
        // to ensure we fail as fast as possible.
        onJoinValidators.stream().forEach(a -> a.accept(node, state));
        if (state.getBlocks().hasGlobalBlock(STATE_NOT_RECOVERED_BLOCK) == false) {
            MembershipAction.ensureMajorVersionBarrier(node.getVersion(), state.getNodes().getMinNodeVersion());
        }
        // try and connect to the node, if it fails, we can raise an exception back to the client...
        transportService.connectToNode(node);

        // validate the join request, will throw a failure if it fails, which will get back to the
        // node calling the join request
        try {
            membership.sendValidateJoinRequestBlocking(node, state, joinTimeout);
        ...
        nodeJoinController.handleJoinRequest(node, callback);
    }
}

public synchronized void handleJoinRequest(final DiscoveryNode node, final MembershipAction.JoinCallback callback) {
    if (electionContext != null) {
        electionContext.addIncomingJoin(node, callback);
        checkPendingJoinsAndElectIfNeeded();
    } else {
        masterService.submitStateUpdateTask("zen-disco-node-join",
            node, ClusterStateTaskConfig.build(Priority.URGENT),
            joinTaskExecutor, new JoinTaskListener(callback, logger));
    }
}
```

handleJoinRequest() 中，如果当前节点不是候选 Master 节点（electionContext 为 null，未在进行选举）则只是进行状态更新，即 else 部分。这里我们主要看 Master 候选逻辑。首先，通过 addIncomingJoin 将收到的 Join 请求加入 electionContext 结构的一个 Join 累加器中：
```java
public synchronized void addIncomingJoin(DiscoveryNode node, MembershipAction.JoinCallback callback) {
    ensureOpen();
    joinRequestAccumulator.computeIfAbsent(node, n -> new ArrayList<>()).add(callback);
}
```

添加完成之后，在 checkPendingJoinsAndElectIfNeeded() 中进行判断是否已收到足够多的 master 角色节点的 Join 请求（在首次进入 waitToBeElectedAsMaster() 时也做了此判断检测，见上）。如果尚未达到，不做任何处理继续等待，如果已达到，则进入 closeAndBecomeMaster()，关闭选举标识（一旦关闭，将不再进行选举及接收任何 Join 请求），再通过 electionFinishedListener 关闭选举现场，最后通过 master service 发布集群状态更新、以及后续启动节点监测等，完成成为 Leader 全过程。
```java
public synchronized void closeAndBecomeMaster() {
    ...
    innerClose();

    Map<DiscoveryNode, ClusterStateTaskListener> tasks = getPendingAsTasks();
    final String source = "zen-disco-elected-as-master ([" + tasks.size() + "] nodes joined)";

    tasks.put(BECOME_MASTER_TASK, (source1, e) -> {}); // noop listener, the election finished listener determines result
    tasks.put(FINISH_ELECTION_TASK, electionFinishedListener);
    masterService.submitStateUpdateTasks(source, tasks, ClusterStateTaskConfig.build(Priority.URGENT), joinTaskExecutor);
}
```

## 加入集群
回到选举主函数 innerJoinCluster() 中。上面讲述的是本地节点成为 Leader 的流程，而如果当前节点没有被选中为 Leader，则会进入“加入集群”的流程，即执行 joinElectedMaster()，joinElectedMaster 逻辑简单，在限定重试次数范围内通过多次发送 Join 请求以加入集群：
```java
/**
 * Join a newly elected master.
 *
 * @return true if successful
 */
private boolean joinElectedMaster(DiscoveryNode masterNode) {
    try {
        // first, make sure we can connect to the master
        transportService.connectToNode(masterNode);
    } catch (Exception e) {
    ...
    int joinAttempt = 0; // we retry on illegal state if the master is not yet ready
    while (true) {
        try {
            logger.trace("joining master {}", masterNode);
            membership.sendJoinRequestBlocking(masterNode, transportService.getLocalNode(), joinTimeout);
            return true;
        } catch (Exception e) {
            final Throwable unwrap = ExceptionsHelper.unwrapCause(e);
            if (unwrap instanceof NotMasterException) {
                if (++joinAttempt == this.joinRetryAttempts) {
                    logger.info("failed to send join request to master [{}], reason [{}], tried [{}] times", masterNode, ExceptionsHelper.detailedMessage(e), joinAttempt);
                    return false;
                } else {
                    ...
        }

        try {
            Thread.sleep(this.joinRetryDelay.millis());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

## 节点失效监测
最后再说下节点失效监测逻辑。如前面所分析，无论节点被选为主节点还是未被选中，到最后都会通过 masterService.submitStateUpdateTasks() 提交 statue update 任务，而这些任务最终会通过 MasterService 类的 runTasks() 执行，runTasks 中最终又调用了最初 Node 初始化时注册的 publish() 集群状态发布函数。在 publish 的一系列调用后，最终会启动节点/Master 故障检测：
```java
// update failure detection only after the state has been updated to prevent race condition with handleLeaveRequest
// and handleNodeFailure as those check the current state to determine whether the failure is to be handled by this node
if (newClusterState.nodes().isLocalNodeElectedMaster()) {
    // update the set of nodes to ping
    nodesFD.updateNodesAndPing(newClusterState);
} else {
    // check to see that we monitor the correct master of the cluster
    if (masterFD.masterNode() == null || !masterFD.masterNode().equals(newClusterState.nodes().getMasterNode())) {
        masterFD.restart(newClusterState.nodes().getMasterNode(),
            "new cluster state received and we are monitoring the wrong master [" + masterFD.masterNode() + "]");
    }
}
```

其中，如果当前节点的角色是选举出的主节点，则启用 nodesFD 监测普通节点状态，如果当前节点角色是普通节点，则启用 masterFD 监测 Master 节点状态。nodesFD 和 masterFD 都是通过 threadPool.schedule() 启动定时任务进行检测，每 1s 发一次 ping 请求，如果大于等于 3 次（默认值）响应异常则认为节点故障。

当 Master 节点异常时，进行重新选举 rejoin 操作：
```java
private void handleMasterGone(final DiscoveryNode masterNode, final Throwable cause, final String reason) {
    if (lifecycleState() != Lifecycle.State.STARTED) {
        // not started, ignore a master failure
        return;
    }
    if (localNodeMaster()) {
        // we might get this on both a master telling us shutting down, and then the disconnect failure
        return;
    }

    logger.info(() -> new ParameterizedMessage("master_left [{}], reason [{}]", masterNode, reason), cause);

    synchronized (stateMutex) {
        if (localNodeMaster() == false && masterNode.equals(committedState.get().nodes().getMasterNode())) {
            // flush any pending cluster states from old master, so it will not be set as master again
            pendingStatesQueue.failAllStatesAndClear(new ElasticsearchException("master left [{}]", reason));
            rejoin("master left (reason = " + reason + ")");
        }
    }
}
```

当普通节点异常时，进行节点移除操作：
```java
private void handleNodeFailure(final DiscoveryNode node, final String reason) {
    if (lifecycleState() != Lifecycle.State.STARTED) {
        // not started, ignore a node failure
        return;
    }
    if (!localNodeMaster()) {
        // nothing to do here...
        return;
    }
    removeNode(node, "zen-disco-node-failed", reason);
}
```

在节点移除的过程中，如果判断剩余的 master 角色节点数量不足，则当前节点会放弃 Leader 角色，进入 rejoin 进行重新选举，以避免同时存在多主造成脑裂现象。否则则直接进入数据 reroute、unassign 等 deassociateDeadNodes 操作。见 NodeRemovalClusterStateTaskExecutor
类中 execute 操作：
```java
@Override
public ClusterTasksResult<Task> execute(final ClusterState currentState, final List<Task> tasks) throws Exception {
    ...
    final ClusterTasksResult.Builder<Task> resultBuilder = ClusterTasksResult.<Task>builder().successes(tasks);
    if (electMasterService.hasEnoughMasterNodes(remainingNodesClusterState.nodes()) == false) {
        final int masterNodes = electMasterService.countMasterNodes(remainingNodesClusterState.nodes());
        rejoin.accept(LoggerMessageFormat.format("not enough master nodes (has [{}], but needed [{}])",
                                                 masterNodes, electMasterService.minimumMasterNodes()));
        return resultBuilder.build(currentState);
    } else {
        return resultBuilder.build(allocationService.deassociateDeadNodes(remainingNodesClusterState, true, describeTasks(tasks)));
    }
}
```

