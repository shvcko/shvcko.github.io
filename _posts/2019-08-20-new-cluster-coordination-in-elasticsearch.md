---
layout: post
title: "Elasticsearch 新版选主流程"
categories: blog
tags: ["Elasticsearch", "Raft", "es 选主流程", "分布式一致性", "源码解析"]
---

在 Elasticsearch 7.0 版本中，Elasticsearch 采用了一个新的集群协调模块（Coordinator）代替了旧版 Zen Discovery，对选主算法 Bully 也进行了更换。根据 ES 的说明，新协调模块提供了如下优势：1、免去了 Zen Discovery 的 discovery.zen.minimum_master_nodes 配置，es 会自己选择可以形成仲裁的节点，用户只需配置一个初始 master 节点列表即可。也即集群扩容或缩容的过程中，不要再担心遗漏或配错 discovery.zen.minimum_master_nodes 配置了；2、新版 Leader 选举速度极大快于旧版。在旧版 Zen Discovery 中，每个节点都需要先通过 3 轮的 ZenPing 才能完成节点发现和 Leader 选举，而新版的算法通常只需要在 100ms 以内；3、修复了 Zen Discovery 下的疑难问题，如“重复的网络分区可能导致群集状态更新丢失”问题。目前业界的分布式一致性算法的理论和工程实现都已经很成熟，出于 es 新旧版本之间平稳升级以及 es 实际应用中一些情况考虑，es 官方并没有直接采用现成开源的第三方一致性算法库，而是结合 es 实际情况自己进行了开发，但其实现的思想还是参考了 Raft 一致性算法，和其基本类似。比如角色一样分为 Leader、Candidate、Follower；将时间划分为一个个任意长度的 term（以连续整数编号）每个 term 起始于leader 选举，可以用于过期信息检测；选举 Leader 时为保证可进展性，各节点采用了随机超时启动选举等。es 社区对新协调模块实现的记录可以见 issue-32006（https://github.com/elastic/elasticsearch/issues/32006），下面对新版的 es 选主流程源码做一下分析。

## Leader 选举
和旧版 Zen Discovery 模块一样，新版节点选主流程的入口函数也在 startInitialJoin()。一个节点在启动时，完成本地初始化工作后，就会调用 Discovery 的 startInitialJoin 进行 Leader 选举操作：
```java
@Override
public void startInitialJoin() {
    synchronized (mutex) {
        becomeCandidate("startInitialJoin");
    }
    clusterBootstrapService.scheduleUnconfiguredBootstrap();
}
```

节点要参与选举，首先要使自己变为候选人也即 Candidate 角色，选举完成后才最终确认自己是成为 Leader 还是 Follower。所以在 startInitialJoin() 中，首先执行的就是 becomeCandidate()。becomeCandidate 主要是做环境初始化工作，停掉或去除任何不符合 Candidate 角色的操作和状态，创建符合新的候选人角色环境。而具体选举流程代码则在后面 clusterBootstrapService.scheduleUnconfiguredBootstrap 中。scheduleUnconfiguredBootstrap() 首先会对节点进行判断，如果节点类型不是 master 节点，就直接返回，不再继续进行。另外，如果集群中节点类型都是旧模式（Zen Discovery）节点，新选举也不会继续进行：
```java
void scheduleUnconfiguredBootstrap() {
    ...
    if (transportService.getLocalNode().isMasterNode() == false) {
        return;
    }
    ...
    transportService.getThreadPool().scheduleUnlessShuttingDown(unconfiguredBootstrapTimeout, Names.GENERIC, new Runnable() {
        @Override
        public void run() {
            final Set<DiscoveryNode> discoveredNodes = getDiscoveredNodes();
            final List<DiscoveryNode> zen1Nodes = discoveredNodes.stream().filter(Coordinator::isZen1Node).collect(Collectors.toList());
            if (zen1Nodes.isEmpty()) {
                logger.debug("performing best-effort cluster bootstrapping with {}", discoveredNodes);
                startBootstrap(discoveredNodes, emptyList());
            } else {
                logger.info("avoiding best-effort cluster bootstrapping due to discovery of pre-7.0 nodes {}", zen1Nodes);
            }
        }
        ...
    });
}
```

在 scheduleUnconfiguredBootstrap() 中调用 startBootstrap() 后，再调用 doBootstrap()。
```java
private void doBootstrap(VotingConfiguration votingConfiguration) {
    ...
    try {
        votingConfigurationConsumer.accept(votingConfiguration);
    } catch (Exception e) {
        ...
        transportService.getThreadPool().scheduleUnlessShuttingDown(TimeValue.timeValueSeconds(10), Names.GENERIC,
            new Runnable() {
                @Override
                public void run() {
                    doBootstrap(votingConfiguration);
                }
                ...
            }
        );
    }
}
```

在 doBootstrap() 中，使用 Lambda 表达式 votingConfigurationConsumer.accept() 继续执行。如果期间出现异常，会 catch 住异常之后重复执行。votingConfigurationConsumer Lambda 表达式在节点初始化时传入，具体对应函数为 Coordinator 的 setInitialConfiguration()：
```java
public boolean setInitialConfiguration(final VotingConfiguration votingConfiguration) {
    synchronized (mutex) {
        ...
        coordinationState.get().setInitialState(ClusterState.builder(currentState).metaData(metaDataBuilder).build());
        ...
        preVoteCollector.update(getPreVoteResponse(), null); // pick up the change to last-accepted version
        startElectionScheduler();
        return true;
    }
}
```

setInitialConfiguration 在做了诸多初始条件判断（是否 master 节点，投票节点是否包含本地节点，投票节点能否形成多数决议等）后，初始化 coordinationState 及 preVoteCollector 内的 response 状态，调用 startElectionScheduler() 执行选举。

startElectionScheduler() 最终会调用至 PreVoteCollector 的 start()，在 start() 里，当前节点会给所有当前非旧版的节点发送 pre_vote 请求，此请求类似于竞选通告，告知所有其他符合条件的节点，我已经开始竞选 leader 了：
```java
void start(final Iterable<DiscoveryNode> broadcastNodes) {
    ...
    broadcastNodes.forEach(n -> transportService.sendRequest(n, REQUEST_PRE_VOTE_ACTION_NAME, preVoteRequest,
        new TransportResponseHandler<PreVoteResponse>() {
            ...

            @Override
            public void handleResponse(PreVoteResponse response) {
                handlePreVoteResponse(response, n);
            }

            @Override
            public void handleException(TransportException exp) {
                logger.debug(new ParameterizedMessage("{} failed", this), exp);
            }

            ...
        }));
}
```

pre_vote 请求发出后，要看 2 部分逻辑，一部分是收到请求的节点如何处理，另一部分是本节点如何处理请求响应。根据逻辑先后顺序，我们先看接收节点如何处理请求。

PreVoteCollector 中注册了 pre_vote 请求（internal:cluster/request_pre_vote）的处理函数，只是一个 Lambda 表达式：
```java
transportService.registerRequestHandler(REQUEST_PRE_VOTE_ACTION_NAME, Names.GENERIC, false, false,
    PreVoteRequest::new,
    (request, channel, task) -> channel.sendResponse(handlePreVoteRequest(request)));
```

在 Lambda 中，接收节点先调用 handlePreVoteRequest(request) 处理请求，然后直接返回处理结果给发送端。handlePreVoteRequest() 相关代码如下：
```java
private PreVoteResponse handlePreVoteRequest(final PreVoteRequest request) {
    updateMaxTermSeen.accept(request.getCurrentTerm());

    Tuple<DiscoveryNode, PreVoteResponse> state = this.state;
    assert state != null : "received pre-vote request before fully initialised";

    final DiscoveryNode leader = state.v1();
    final PreVoteResponse response = state.v2();

    if (leader == null) {
        return response;
    }

    if (leader.equals(request.getSourceNode())) {
        // This is a _rare_ case where our leader has detected a failure and stepped down, but we are still a follower. It's possible
        // that the leader lost its quorum, but while we're still a follower we will not offer joins to any other node so there is no
        // major drawback in offering a join to our old leader. The advantage of this is that it makes it slightly more likely that the
        // leader won't change, and also that its re-election will happen more quickly than if it had to wait for a quorum of followers
        // to also detect its failure.
        return response;
    }

    throw new CoordinationStateRejectedException("rejecting " + request + " as there is already a leader");
}
```

首先通过 updateMaxTermSeen 进行接收到请求的 term 进行检测，updateMaxTermSeen 也是一个 Lambda 表达式，具体实现对应 updateMaxTermSeen()：
```java
private void updateMaxTermSeen(final long term) {
    synchronized (mutex) {
        maxTermSeen = Math.max(maxTermSeen, term);
        final long currentTerm = getCurrentTerm();
        if (mode == Mode.LEADER && maxTermSeen > currentTerm) {
            // Bump our term. However if there is a publication in flight then doing so would cancel the publication, so don't do that
            // since we check whether a term bump is needed at the end of the publication too.
            if (publicationInProgress()) {
                logger.debug("updateMaxTermSeen: maxTermSeen = {} > currentTerm = {}, enqueueing term bump", maxTermSeen, currentTerm);
            } else {
                try {
                    logger.debug("updateMaxTermSeen: maxTermSeen = {} > currentTerm = {}, bumping term", maxTermSeen, currentTerm);
                    ensureTermAtLeast(getLocalNode(), maxTermSeen);
                    startElection();
                } catch (Exception e) {
                    logger.warn(new ParameterizedMessage("failed to bump term to {}", maxTermSeen), e);
                    becomeCandidate("updateMaxTermSeen");
                }
            }
        }
    }
}
```

其主要逻辑是将请求的 term 与当前节点接收过的最大的 term 进行比较，如果当前节点角色是 Leader，并且新接收 request 的 term 大于当前节点最大的 term，则意味着当前自身节点信息可能已经过期，其他节点已经开始竞选 Leader 了。所以先通过 ensureTermAtLeast() 立即停止自己的 Leader 身份，转变为 Candidate，并重新参与选举。

如果确认 term 没有问题，则继续 handlePreVoteRequest 的后续逻辑：如果当前节点还未收到有节点当选为 Leader 的信息（if (leader == null)），则直接返回 response，表示支持请求节点选举；已经有 leader 信息，但和请求节点相同，也返回 response 表示同意。其他情况下，都拒绝请求节点的竞选（throw new CoordinationStateRejectedException），因为已经有 Leader 节点。

以上是 pre_vote 请求处理，下面再看下本地竞选节点如何处理 pre_vote 请求响应：
```java
private void handlePreVoteResponse(final PreVoteResponse response, final DiscoveryNode sender) {
    ...
    updateMaxTermSeen.accept(response.getCurrentTerm());

    if (response.getLastAcceptedTerm() > clusterState.term()
        || (response.getLastAcceptedTerm() == clusterState.term()
        && response.getLastAcceptedVersion() > clusterState.getVersionOrMetaDataVersion())) {
        logger.debug("{} ignoring {} from {} as it is fresher", this, response, sender);
        return;
    }

    preVotesReceived.add(sender);
    final VoteCollection voteCollection = new VoteCollection();
    preVotesReceived.forEach(voteCollection::addVote);

    if (isElectionQuorum(voteCollection, clusterState) == false) {
        logger.debug("{} added {} from {}, no quorum yet", this, response, sender);
        return;
    }

    if (electionStarted.compareAndSet(false, true) == false) {
        logger.debug("{} added {} from {} but election has already started", this, response, sender);
        return;
    }

    logger.debug("{} added {} from {}, starting election", this, response, sender);
    startElection.run();
}
```

在 handlePreVoteResponse() 中，首先也是对响应信息的 term 合法性进行判断，逻辑同上。如果响应的 term 大于当前节点 term，或者 term 相同但集群状态信息版本更新，则表明响应节点状态更新，当前节点不做任何处理。否则，响应节点作为投赞成自己选举一票加入到 voteCollection 中。如果投票尚未超过多数也不做任何处理，如果已超过多数，则进入成为 Leader 流程，执行startElection.run()。

startElection Lambda 表达式执行 startElection() 方法，发送 StartJoinRequest 请求：
```java
private void startElection() {
    synchronized (mutex) {
        // The preVoteCollector is only active while we are candidate, but it does not call this method with synchronisation, so we have
        // to check our mode again here.
        if (mode == Mode.CANDIDATE) {
            if (electionQuorumContainsLocalNode(getLastAcceptedState()) == false) {
                logger.trace("skip election as local node is not part of election quorum: {}",
                    getLastAcceptedState().coordinationMetaData());
                return;
            }

            final StartJoinRequest startJoinRequest
                = new StartJoinRequest(getLocalNode(), Math.max(getCurrentTerm(), maxTermSeen) + 1);
            logger.debug("starting election with {}", startJoinRequest);
            getDiscoveredNodes().forEach(node -> {
                if (isZen1Node(node) == false) {
                    joinHelper.sendStartJoinRequest(startJoinRequest, node);
                }
            });
        }
    }
}
```

StartJoin 请求不同于 Join 请求，Join 请求时某个节点请求加入 Leader 所在集群，而 StartJoin 则是 Leader 通知其他节点加入自己集群。发送 StartJoinRequest 请求代码如下：
```java
public void sendStartJoinRequest(final StartJoinRequest startJoinRequest, final DiscoveryNode destination) {
    ...
    transportService.sendRequest(destination, START_JOIN_ACTION_NAME,
        startJoinRequest, new TransportResponseHandler<Empty>() {
            ...
            @Override
            public void handleResponse(Empty response) {
                logger.debug("successful response to {} from {}", startJoinRequest, destination);
            }

            @Override
            public void handleException(TransportException exp) {
                logger.debug(new ParameterizedMessage("failure in response to {} from {}", startJoinRequest, destination), exp);
            }
            ...
        });
}
```

可见其 handleResponse 不做任何处理。由其对应处理函数可知，StartJoin 请求只起一个通知作用，通告其他节点可以发送 Join 请求了：
```java
transportService.registerRequestHandler(START_JOIN_ACTION_NAME, Names.GENERIC, false, false,
    StartJoinRequest::new,
    (request, channel, task) -> {
        final DiscoveryNode destination = request.getSourceNode();
        sendJoinRequest(destination, Optional.of(joinLeaderInTerm.apply(request)));
        channel.sendResponse(Empty.INSTANCE);
    });
```

其他节点发送 Join 请求时，先通过 joinLeaderInTerm.apply() 调用 joinLeaderInTerm()，构造 Join 请求，获取 term，以及转为 Candidate 角色等，见下：
```java
private Join joinLeaderInTerm(StartJoinRequest startJoinRequest) {
    synchronized (mutex) {
        logger.debug("joinLeaderInTerm: for [{}] with term {}", startJoinRequest.getSourceNode(), startJoinRequest.getTerm());
        final Join join = coordinationState.get().handleStartJoin(startJoinRequest);
        lastJoin = Optional.of(join);
        peerFinder.setCurrentTerm(getCurrentTerm());
        if (mode != Mode.CANDIDATE) {
            becomeCandidate("joinLeaderInTerm"); // updates followersChecker and preVoteCollector
        } else {
            followersChecker.updateFastResponseState(getCurrentTerm(), mode);
            preVoteCollector.update(getPreVoteResponse(), null);
        }
        return join;
    }
}
```

最后通过 sendJoinRequest() 发送 Join 请求：
```java
public void sendJoinRequest(DiscoveryNode destination, Optional<Join> optionalJoin, Runnable onCompletion) {
    ...
    final JoinRequest joinRequest = new JoinRequest(transportService.getLocalNode(), optionalJoin);
    final Tuple<DiscoveryNode, JoinRequest> dedupKey = Tuple.tuple(destination, joinRequest);
    if (pendingOutgoingJoins.add(dedupKey)) {
        ...
        } else {
            actionName = JOIN_ACTION_NAME;
            transportRequest = joinRequest;
        }
        transportService.sendRequest(destination, actionName, transportRequest,
            TransportRequestOptions.builder().withTimeout(joinTimeout).build(),
            new TransportResponseHandler<Empty>() {
                ...
                @Override
                public void handleResponse(Empty response) {
                    pendingOutgoingJoins.remove(dedupKey);
                    logger.debug("successfully joined {} with {}", destination, joinRequest);
                    onCompletion.run();
                    lastFailedJoinAttempt.set(null);
                }

                @Override
                public void handleException(TransportException exp) {
                    pendingOutgoingJoins.remove(dedupKey);
                    logger.info(() -> new ParameterizedMessage("failed to join {} with {}", destination, joinRequest), exp);
                    onCompletion.run();
                    FailedJoinAttempt attempt = new FailedJoinAttempt(destination, joinRequest, exp);
                    attempt.logNow();
                    lastFailedJoinAttempt.set(attempt);
                }
                ...
            });
    } else {
        ...
    }
}
```

在 Join 请求的接收方面，则是通过 Lambda 表达式调用 handleJoinRequest() 处理：
```java
transportService.registerRequestHandler(JOIN_ACTION_NAME, ThreadPool.Names.GENERIC, false, false, JoinRequest::new,
    (request, channel, task) -> joinHandler.accept(request, transportJoinCallback(request, channel)));
```

handleJoinRequest() 中判断如果是当前节点在竞选 Leader，则在对版本等做一系列检测后，发送JoinValidate 请求验证 Join 合法性：
```java
private void handleJoinRequest(JoinRequest joinRequest, JoinHelper.JoinCallback joinCallback) {
    ...
    if (stateForJoinValidation.nodes().isLocalNodeElectedMaster()) {
        onJoinValidators.forEach(a -> a.accept(joinRequest.getSourceNode(), stateForJoinValidation));
        if (stateForJoinValidation.getBlocks().hasGlobalBlock(STATE_NOT_RECOVERED_BLOCK) == false) {
            // we do this in a couple of places including the cluster update thread. This one here is really just best effort
            // to ensure we fail as fast as possible.
            JoinTaskExecutor.ensureMajorVersionBarrier(joinRequest.getSourceNode().getVersion(),
                stateForJoinValidation.getNodes().getMinNodeVersion());
        }
        sendValidateJoinRequest(stateForJoinValidation, joinRequest, joinCallback);

    } else {
        processJoinRequest(joinRequest, joinCallback);
    }
}
```

在 sendValidateJoinRequest() 的请求响应函数中，调用 processJoinRequest 对 Join 进行处理。在 processJoinRequest 中 ，调用 handleJoin()：
```java
private void processJoinRequest(JoinRequest joinRequest, JoinHelper.JoinCallback joinCallback) {
    final Optional<Join> optionalJoin = joinRequest.getOptionalJoin();
    synchronized (mutex) {
        final CoordinationState coordState = coordinationState.get();
        final boolean prevElectionWon = coordState.electionWon();

        optionalJoin.ifPresent(this::handleJoin);
        joinAccumulator.handleJoinRequest(joinRequest.getSourceNode(), joinCallback);

        if (prevElectionWon == false && coordState.electionWon()) {
            becomeLeader("handleJoinRequest");
        }
    }
}
```

handleJoin() 中，还是对 term 再确认判断（ensureTermAtLeast()），如无问题则调用 coordinationState.get().handleJoin(join)：
```java
private void handleJoin(Join join) {
    synchronized (mutex) {
        ensureTermAtLeast(getLocalNode(), join.getTerm()).ifPresent(this::handleJoin);

        if (coordinationState.get().electionWon()) {
            // If we have already won the election then the actual join does not matter for election purposes, so swallow any exception
            final boolean isNewJoin = handleJoinIgnoringExceptions(join);

            // If we haven't completely finished becoming master then there's already a publication scheduled which will, in turn,
            // schedule a reconfiguration if needed. It's benign to schedule a reconfiguration anyway, but it might fail if it wins the
            // race against the election-winning publication and log a big error message, which we can prevent by checking this here:
            final boolean establishedAsMaster = mode == Mode.LEADER && getLastAcceptedState().term() == getCurrentTerm();
            if (isNewJoin && establishedAsMaster && publicationInProgress() == false) {
                scheduleReconfigurationIfNeeded();
            }
        } else {
            coordinationState.get().handleJoin(join); // this might fail and bubble up the exception
        }
    }
}
```

coordinationState 的 handleJoin 做各种条件判断及最终的 Leader 选举是否成功判断，如果 isElectionQuorum 则竞选成功：
```java
public boolean handleJoin(Join join) {
    ...
    boolean added = joinVotes.addVote(join.getSourceNode());
    boolean prevElectionWon = electionWon;
    electionWon = isElectionQuorum(joinVotes);
    assert !prevElectionWon || electionWon; // we cannot go from won to not won
    logger.debug("handleJoin: added join {} from [{}] for election, electionWon={} lastAcceptedTerm={} lastAcceptedVersion={}", join,
        join.getSourceNode(), electionWon, lastAcceptedTerm, getLastAcceptedVersion());

    if (electionWon && prevElectionWon == false) {
        logger.debug("handleJoin: election won in term [{}] with {}", getCurrentTerm(), joinVotes);
        lastPublishedVersion = getLastAcceptedVersion();
    }
    return added;
}
```

判断是否赢得选举后（electionWon），回到 processJoinRequest()，通过 joinAccumulator.handleJoinRequest()，如果节点是 Leader，则发布状态更新：
```java
@Override
public void handleJoinRequest(DiscoveryNode sender, JoinCallback joinCallback) {
    final JoinTaskExecutor.Task task = new JoinTaskExecutor.Task(sender, "join existing leader");
    masterService.submitStateUpdateTask("node-join", task, ClusterStateTaskConfig.build(Priority.URGENT),
        joinTaskExecutor, new JoinTaskListener(task, joinCallback));
}
```

最后，判断如果节点之前的状态未赢得选举，而这次赢得选举，则执行 becomeLeader() 成为 Leader。

