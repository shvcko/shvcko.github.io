---
layout: post
title: "Elasticsearch Clients 内部机制"
categories: blog
tags: ["Elasticsearch", "es 客户端内部机制", "源码解析"]
---

本文试图通过源码分析阐释 Elasticsearch Clients 如下内部机制问题：
* es 集群服务端异常，将如何影响客户端的请求？es 客户端能否自动重试？
* es 客户端对多地址如何管理，新增地址或删除地址如何感知？
* es 客户端负载均衡采用的是何种策略？
* 对于 es 集群某个节点的掉线，es 客户端发送请求时如何感知，由会采用何种解决策略？
* es 客户端请求是否用的是长连接，能否继续提升性能？
* es sniff 机制作用？

这些问题是业务方在使用 Elasticsearch Clients 时的常见问题。Elasticsearch Clients 全部类型可见：https://www.elastic.co/guide/en/elasticsearch/client/index.html。由于实际工作中 Java REST Client 和 Java API（TransportClient）使用较多，因此此处暂只分析这两种类型客户端源码。

—备注：分析的 Elasticsearch Clients 版本为 6.5.x。

## 1、添加多地址
不论是 Elasticsearch Java REST Client 还是 Elasticsearch Java API，在使用时，都允许用户添加多个集群访问地址。

Java REST Client
```java
RestClient restClient = RestClient.builder(
    new HttpHost("host1", 9200, "http"),
    new HttpHost("host2", 9200, "http")).build();
```

Java API
```java
TransportClient client = new PreBuiltTransportClient(Settings.EMPTY)
        .addTransportAddress(new TransportAddress(InetAddress.getByName("host1"), 9300))
        .addTransportAddress(new TransportAddress(InetAddress.getByName("host2"), 9300));
```

添加之后，Elasticsearch 在 Client 内部将其保存在一个 List 列表当中。其中，Rest 客户端中保存为了 Node 结构列表；而 Transport 客户端内部则转为了 DiscoveryNode 列表。

Java REST Client
```java
...
List<Node> nodes = new ArrayList<>(hosts.length);
for (HttpHost host : hosts) {
    nodes.add(new Node(host));
}
...
```

Java API
```java
...
List<DiscoveryNode> builder = new ArrayList<>(listedNodes);
for (TransportAddress transportAddress : filtered) {
    DiscoveryNode node = new DiscoveryNode("#transport#-" + tempNodeIdGenerator.incrementAndGet(),
            transportAddress, Collections.emptyMap(), Collections.emptySet(), minCompatibilityVersion);
    logger.debug("adding address [{}]", node);
    builder.add(node);
}
listedNodes = Collections.unmodifiableList(builder);
...
```

## 2、Elasticsearch Clients 探活机制
### Java API 探活机制

Java API（TransportClient） 有定时的（采样）探活机制，在 TransportClient 初始化时，会初始化 TransportClientNodesService，在其构造函数中，就会启动采样定时器：
```java
TransportClientNodesService(Settings settings, TransportService transportService,
                                   ThreadPool threadPool, TransportClient.HostFailureListener hostFailureListener) {
    ...
    this.nodesSamplerInterval = TransportClient.CLIENT_TRANSPORT_NODES_SAMPLER_INTERVAL.get(this.settings);
    ...
    if (TransportClient.CLIENT_TRANSPORT_SNIFF.get(this.settings)) {
        this.nodesSampler = new SniffNodesSampler();
    } else {
        this.nodesSampler = new SimpleNodeSampler();
    }
    ...
    this.nodesSamplerFuture = threadPool.schedule(nodesSamplerInterval, ThreadPool.Names.GENERIC, new ScheduledNodeSampler());
}
```

其中 CLIENT_TRANSPORT_NODES_SAMPLER_INTERVAL 配置值默认为 5s，也即每隔 5s TransportClient 对所有节点探测一次。Sampler 类分两种，简单采样（SimpleNodeSampler）类和嗅探采样（SniffNodesSampler）类，具体采用哪种看客户端 CLIENT_TRANSPORT_SNIFF（client.transport.sniff）配置。另外，除定时触发的检测外，每次新节点的加入（addTransportAddresses）或某个节点的移除（removeTransportAddress）也都会触发一次采样检测。
```java
public TransportClientNodesService addTransportAddresses(TransportAddress... transportAddresses) {
    ...
        nodesSampler.sample();
    ...
}

public TransportClientNodesService removeTransportAddress(TransportAddress transportAddress) {
    ...
        nodes = Collections.unmodifiableList(nodesBuilder);
    ...
}
```

再看下采样的具体流程：
```java
class SimpleNodeSampler extends NodeSampler {

    @Override
    protected void doSample() {
        HashSet<DiscoveryNode> newNodes = new HashSet<>();
        ArrayList<DiscoveryNode> newFilteredNodes = new ArrayList<>();
        for (DiscoveryNode listedNode : listedNodes) {
            try (Transport.Connection connection = transportService.openConnection(listedNode, LISTED_NODES_PROFILE)){
                final PlainTransportFuture<LivenessResponse> handler = new PlainTransportFuture<>(
                    new FutureTransportResponseHandler<LivenessResponse>() {
                        @Override
                        public LivenessResponse read(StreamInput in) throws IOException {
                            LivenessResponse response = new LivenessResponse();
                            response.readFrom(in);
                            return response;
                        }
                    });
                transportService.sendRequest(connection, TransportLivenessAction.NAME, new LivenessRequest(),
                    TransportRequestOptions.builder().withType(TransportRequestOptions.Type.STATE).withTimeout(pingTimeout).build(),
                    handler);
                final LivenessResponse livenessResponse = handler.txGet();
                if (!ignoreClusterName && !clusterName.equals(livenessResponse.getClusterName())) {
                    logger.warn("node {} not part of the cluster {}, ignoring...", listedNode, clusterName);
                    newFilteredNodes.add(listedNode);
                } else {
                    // use discovered information but do keep the original transport address,
                    // so people can control which address is exactly used.
                    DiscoveryNode nodeWithInfo = livenessResponse.getDiscoveryNode();
                    newNodes.add(new DiscoveryNode(nodeWithInfo.getName(), nodeWithInfo.getId(), nodeWithInfo.getEphemeralId(),
                        nodeWithInfo.getHostName(), nodeWithInfo.getHostAddress(), listedNode.getAddress(),
                        nodeWithInfo.getAttributes(), nodeWithInfo.getRoles(), nodeWithInfo.getVersion()));
                }
            } catch (ConnectTransportException e) {
                logger.debug(() -> new ParameterizedMessage("failed to connect to node [{}], ignoring...", listedNode), e);
                hostFailureListener.onNodeDisconnected(listedNode, e);
            } catch (Exception e) {
                logger.info(() -> new ParameterizedMessage("failed to get node info for {}, disconnecting...", listedNode), e);
            }
        }

        nodes = establishNodeConnections(newNodes);
        filteredNodes = Collections.unmodifiableList(newFilteredNodes);
    }
}
```

SimpleNodeSampler 中，其探测思路就是给所有在列表中的节点（参照“添加多地址”部分）发送 liveness 请求，如果正常响应且集群名称等信息能核对上，则为正常的节点，并加入到正常节点列表中备用。如果连接/响应异常，则输出 debug 日志且不再做其他处理（hostFailureListener.onNodeDisconnected() 目前未做任何处理）。如果集群名称不对，则加入过滤列表，目前暂不使用。另外，加入正常列表中的节点会调用 establishNodeConnections() 给未连接的节点建立连接，方便后续发送请求时直接使用。

```java
    class SniffNodesSampler extends NodeSampler {

        @Override
        protected void doSample() {
            ...
                        @Override
                        protected void doRun() throws Exception {
                            ...
                            transportService.sendRequest(pingConnection, ClusterStateAction.NAME,
                                Requests.clusterStateRequest().clear().nodes(true).local(true),
                                TransportRequestOptions.builder().withType(TransportRequestOptions.Type.STATE)
                                    .withTimeout(pingTimeout).build(),
                                new TransportResponseHandler<ClusterStateResponse>() {
                                    ...
                                    @Override
                                    public void handleResponse(ClusterStateResponse response) {
                                        clusterStateResponses.put(nodeToPing, response);
                                        onDone();
                                    }
                                    ...
            }

            HashSet<DiscoveryNode> newNodes = new HashSet<>();
            HashSet<DiscoveryNode> newFilteredNodes = new HashSet<>();
            for (Map.Entry<DiscoveryNode, ClusterStateResponse> entry : clusterStateResponses.entrySet()) {
                if (!ignoreClusterName && !clusterName.equals(entry.getValue().getClusterName())) {
                    logger.warn("node {} not part of the cluster {}, ignoring...",
                            entry.getValue().getState().nodes().getLocalNode(), clusterName);
                    newFilteredNodes.add(entry.getKey());
                    continue;
                }
                for (ObjectCursor<DiscoveryNode> cursor : entry.getValue().getState().nodes().getDataNodes().values()) {
                    newNodes.add(cursor.value);
                }
            }

            nodes = establishNodeConnections(newNodes);
            filteredNodes = Collections.unmodifiableList(new ArrayList<>(newFilteredNodes));
        }
    }
```

SniffNodesSampler 的整体代码较长，因此这里对部分不太相关代码进行了省略。总体流程上，SniffNodesSampler 和 SimpleNodeSampler 类似。不过 SniffNodesSampler 发送探测消息时，发送的是集群状态（ClusterStateAction）请求，因此，收到的响应是集群状态信息，也就包含了集群中所有节点信息。在后面的 for 循环处理响应信息流程中，SniffNodesSampler 取出了所有 data 类型节点，并加入到了可用节点候选列表中，因此后续用户也就可以直接连接 data 类型节点了。

### Java REST Client 探活机制

Java REST Client 没有定时的探活检测机制，它将节点状态的检测结合在了每次用户请求中。当发送用户请求时，如果响应正常，则表明节点状态是 ok 的，如果请求失败，则标记节点为异常。RestClient 处理请求的主要逻辑在 performRequestAsync() 函数：
```java
    private void performRequestAsync(final long startTime, final NodeTuple<Iterator<Node>> nodeTuple, final HttpRequestBase request,
                                     final Set<Integer> ignoreErrorCodes,
                                     final HttpAsyncResponseConsumerFactory httpAsyncResponseConsumerFactory,
                                     final FailureTrackingResponseListener listener) {
        final Node node = nodeTuple.nodes.next();
        ...
        client.execute(requestProducer, asyncResponseConsumer, context, new FutureCallback<HttpResponse>() {
            @Override
            public void completed(HttpResponse httpResponse) {
                try {
                    RequestLogger.logResponse(logger, request, node.getHost(), httpResponse);
                    int statusCode = httpResponse.getStatusLine().getStatusCode();
                    Response response = new Response(request.getRequestLine(), node.getHost(), httpResponse);
                    if (isSuccessfulResponse(statusCode) || ignoreErrorCodes.contains(response.getStatusLine().getStatusCode())) {
                        onResponse(node);
                        if (strictDeprecationMode && response.hasWarnings()) {
                            listener.onDefinitiveFailure(new ResponseException(response));
                        } else {
                            listener.onSuccess(response);
                        }
                    } else {
                        ResponseException responseException = new ResponseException(response);
                        if (isRetryStatus(statusCode)) {
                            //mark host dead and retry against next one
                            onFailure(node);
                            retryIfPossible(responseException);
                        } else {
                            //mark host alive and don't retry, as the error should be a request problem
                            onResponse(node);
                            listener.onDefinitiveFailure(responseException);
                        }
                    }
                } catch(Exception e) {
                    listener.onDefinitiveFailure(e);
                }
            }
            ...
    }
```

其中正常响应的结果处理函数为 onResponse()，如果节点在黑名单中（blacklist），则将其移出：
```java
/**
 * Called after each successful request call.
 * Receives as an argument the host that was used for the successful request.
 */
private void onResponse(Node node) {
    DeadHostState removedHost = this.blacklist.remove(node.getHost());
    if (logger.isDebugEnabled() && removedHost != null) {
        logger.debug("removed [" + node + "] from blacklist");
    }
}
```

响应异常处理函数 onFailure()，将节点加入黑名单列表：
```java
/**
 * Called after each failed attempt.
 * Receives as an argument the host that was used for the failed attempt.
 */
private void onFailure(Node node) {
    while(true) {
        DeadHostState previousDeadHostState =
            blacklist.putIfAbsent(node.getHost(), new DeadHostState(TimeSupplier.DEFAULT));
        if (previousDeadHostState == null) {
            if (logger.isDebugEnabled()) {
                logger.debug("added [" + node + "] to blacklist");
            }
            break;
        }
        if (blacklist.replace(node.getHost(), previousDeadHostState,
                new DeadHostState(previousDeadHostState))) {
            if (logger.isDebugEnabled()) {
                logger.debug("updated [" + node + "] already in blacklist");
            }
            break;
        }
    }
    failureListener.onFailure(node);
}
```

不过，加入黑名单列表之后的节点，也有机会重新被启用。RestClient 主要采用一个重试超时机制来控制。在执行每次请求之前，RestClient 都会判断一下黑名单类别里的节点是否有已经满足重试需求的节点了，如果有则将其取出加入发送请求候选列表。
```java
    void performRequestAsyncNoCatch(Request request, ResponseListener listener) throws IOException {
        ...
        performRequestAsync(startTime, nextNode(), httpRequest, ignoreErrorCodes,
                request.getOptions().getHttpAsyncResponseConsumerFactory(), failureTrackingResponseListener);
    }

    private NodeTuple<Iterator<Node>> nextNode() throws IOException {
        NodeTuple<List<Node>> nodeTuple = this.nodeTuple;
        Iterable<Node> hosts = selectNodes(nodeTuple, blacklist, lastNodeIndex, nodeSelector);
        return new NodeTuple<>(hosts.iterator(), nodeTuple.authCache);
    }

    static Iterable<Node> selectNodes(NodeTuple<List<Node>> nodeTuple, Map<HttpHost, DeadHostState> blacklist,
                                      AtomicInteger lastNodeIndex, NodeSelector nodeSelector) throws IOException {
        /*
         * Sort the nodes into living and dead lists.
         */
        List<Node> livingNodes = new ArrayList<>(nodeTuple.nodes.size() - blacklist.size());
        List<DeadNode> deadNodes = new ArrayList<>(blacklist.size());
        for (Node node : nodeTuple.nodes) {
            DeadHostState deadness = blacklist.get(node.getHost());
            if (deadness == null) {
                livingNodes.add(node);
                continue;
            }
            if (deadness.shallBeRetried()) {
                livingNodes.add(node);
                continue;
            }
            deadNodes.add(new DeadNode(node, deadness));
        }
        ...
    }
```

RestClient 的重试超时机制根据时间间隔来判断，最小时间间隔为 1min，随后逐步增长，一直增加到最长间隔 30min。
```java
/**
 * Indicates whether it's time to retry to failed host or not.
 *
 * @return true if the host should be retried, false otherwise
 */
boolean shallBeRetried() {
    return timeSupplier.nanoTime() - deadUntilNanos > 0;
}

DeadHostState(DeadHostState previousDeadHostState) {
    long timeoutNanos = (long)Math.min(MIN_CONNECTION_TIMEOUT_NANOS * 2 * Math.pow(2, previousDeadHostState.failedAttempts * 0.5 - 1),
            MAX_CONNECTION_TIMEOUT_NANOS);
    this.deadUntilNanos = previousDeadHostState.timeSupplier.nanoTime() + timeoutNanos;
    ...
}

DeadHostState(TimeSupplier timeSupplier) {
    ...
    this.deadUntilNanos = timeSupplier.nanoTime() + MIN_CONNECTION_TIMEOUT_NANOS;
    ...
}
```

## 3、Elasticsearch Clients 请求异常处理机制
### Java API 请求异常处理

Java API 中执行请求的入口为 AbstractClient.java 中 execute() 方法：
```java
/**
 * This is the single execution point of *all* clients.
 */
@Override
public final <Request extends ActionRequest, Response extends ActionResponse, RequestBuilder extends ActionRequestBuilder<Request, Response, RequestBuilder>> void execute(
        Action<Request, Response, RequestBuilder> action, Request request, ActionListener<Response> listener) {
    listener = threadedWrapper.wrap(listener);
    doExecute(action, request, listener);
}
```

其中 doExecute() 为一个抽象方法，由 AbstractClient 的子类（ NodeClient / TransportClient 等）实现。对于 TransportClient 而言，在具体实现中只是简单的调用了它的代理类 TransportProxyClient 对象 proxy 来执行：
```java
@Override
protected <Request extends ActionRequest, Response extends ActionResponse, RequestBuilder extends ActionRequestBuilder<Request, Response, RequestBuilder>> void doExecute(Action<Request, Response, RequestBuilder> action, Request request, ActionListener<Response> listener) {
    proxy.execute(action, request, listener);
}
```

TransportProxyClient execute() 方法的具体代码如下：
```java
public <Request extends ActionRequest, Response extends ActionResponse, RequestBuilder extends
    ActionRequestBuilder<Request, Response, RequestBuilder>> void execute(final Action<Request, Response, RequestBuilder> action,
                                                                          final Request request, ActionListener<Response> listener) {
    final TransportActionNodeProxy<Request, Response> proxy = proxies.get(action);
    assert proxy != null : "no proxy found for action: " + action;
    nodesService.execute((n, l) -> proxy.execute(n, request, l), listener);
}
```

其中 nodesService 的 execute() 主要做请求前的预备工作，如校验连接节点状态、获取具体连接节点等。而发送请求的实际工作在 proxy 的 execute() 中执行。此处有一点需要注意的是执行 proxy.execute() 这块代码的是一个 Lambda 表达式，其实际参数在 nodesService.execute() 中传入，其中第三个参数 l 传入的是一个 RetryListener 对象。相关代码见下：
```java
public <Response> void execute(NodeListenerCallback<Response> callback, ActionListener<Response> listener) {
    final List<DiscoveryNode> nodes = this.nodes;
    if (closed) {
        throw new IllegalStateException("transport client is closed");
    }
    ensureNodesAreAvailable(nodes);
    int index = getNodeNumber();
    RetryListener<Response> retryListener = new RetryListener<>(callback, listener, nodes, index, hostFailureListener);
    DiscoveryNode node = retryListener.getNode(0);
    try {
        callback.doWithNode(node, retryListener);
    } catch (Exception e) {
        try {
            //this exception can't come from the TransportService as it doesn't throw exception at all
            listener.onFailure(e);
        } finally {
            retryListener.maybeNodeFailed(node, e);
        }
    }
}
```

Lambda 表达式 proxy.execute() 中调用的是 TransportService 的 sendRequest，其最后一个参数传入的是 ActionListenerResponseHandler：
```java
public void execute(final DiscoveryNode node, final Request request, final ActionListener<Response> listener) {
    ActionRequestValidationException validationException = request.validate();
    if (validationException != null) {
        listener.onFailure(validationException);
        return;
    }
    transportService.sendRequest(node, action.name(), request, transportOptions,
        new ActionListenerResponseHandler<>(listener, action::newResponse));
}
```

而 TransportService 的 sendRequest 最终调用的是 sendRequestInternal()，其调用关系在 TransportService 对象构造函数中进行了绑定：
```java
...
this.asyncSender = interceptor.interceptSender(this::sendRequestInternal);
...
```

sendRequestInternal() 最终发送用户请求（部分无关代码进行了省略）：
```java
private <T extends TransportResponse> void sendRequestInternal(final Transport.Connection connection, final String action,
                                                               final TransportRequest request,
                                                               final TransportRequestOptions options,
                                                               TransportResponseHandler<T> handler) {
    ...
    try {
        ...
        connection.sendRequest(requestId, action, request, options); // local node optimization happens upstream
    } catch (final Exception e) {
        // usually happen either because we failed to connect to the node
        // or because we failed serializing the message
        ...
                @Override
                protected void doRun() throws Exception {
                    contextToNotify.handler().handleException(sendRequestException);
                }
            });
        ...
}
```

其中 contextToNotify.handler().handleException() 即为处理请求发送异常情况的地方，因此我们只需重点关注这里。由之前分析可知，传入的 handler 为  ActionListenerResponseHandler ，因此我们查看 ActionListenerResponseHandler 类中的 handleException() 实现，其调用的是 listener 的 onFailure()：
```java
@Override
public void handleException(TransportException e) {
    listener.onFailure(e);
}
```

而上上面我们已分析，传给 proxy execute 第三个参数的 listener 是一个 RetryListener，RetryListener 中 onFailure 代码如下：
```java
@Override
public void onFailure(Exception e) {
    Throwable throwable = ExceptionsHelper.unwrapCause(e);
    if (throwable instanceof ConnectTransportException) {
        maybeNodeFailed(getNode(this.i), (ConnectTransportException) throwable);
        int i = ++this.i;
        if (i >= nodes.size()) {
            listener.onFailure(new NoNodeAvailableException("None of the configured nodes were available: " + nodes, e));
        } else {
            try {
                callback.doWithNode(getNode(i), this);
            } catch(final Exception inner) {
                inner.addSuppressed(e);
                // this exception can't come from the TransportService as it doesn't throw exceptions at all
                listener.onFailure(inner);
            }
        }
    } else {
        listener.onFailure(e);
    }
}
```

其主要逻辑，首先判断发生的异常类型是否为 ConnectTransportException。如果不是，则直接调用 listener.onFailure(e) 返回异常信息给用户。如果是，按以下执行：maybeNodeFailed()（目前默认情况下 es 未做任何处理 - 初始化 lambda 表达式为空函数）；++this.i，表示增加序号再取下一个节点，如果 i >= nodes.size 表明 nodes 里面的节点都已取了一遍，此时仍有错误则直接抛出异常，表明已无节点可用。否则，则通过 callback.doWithNode() 调用再次发送请求。

因此，有上述代码可见，在 Java API（TransportClient）中，当发送用户请求异常时，如果发生的异常为节点连接异常（ConnectTransportException），则 TransportClient 会按顺序取下一个节点进行重试，并且一直重试至所有可用节点都试过为止，如果此时仍然发生异常，则发出无可用节点警告。而如果发生的请求异常为非节点连接异常，则 TransportClient 不会进行任何重试或异常额外处理，将直接通过回调接口将信息返回给用户。

### Java REST Client 请求异常处理

Java REST Client 的请求处理入口及后续调用流程比较直观，因此不再逐步细述。我们直接看其处理请求主要逻辑的 RestClient 类 performRequestAsync() 方法：
```java
private void performRequestAsync(final long startTime, final NodeTuple<Iterator<Node>> nodeTuple, final HttpRequestBase request,
                                 final Set<Integer> ignoreErrorCodes,
                                 final HttpAsyncResponseConsumerFactory httpAsyncResponseConsumerFactory,
                                 final FailureTrackingResponseListener listener) {
    final Node node = nodeTuple.nodes.next();
    //we stream the request body if the entity allows for it
    final HttpAsyncRequestProducer requestProducer = HttpAsyncMethods.create(node.getHost(), request);
    final HttpAsyncResponseConsumer<HttpResponse> asyncResponseConsumer =
        httpAsyncResponseConsumerFactory.createHttpAsyncResponseConsumer();
    final HttpClientContext context = HttpClientContext.create();
    context.setAuthCache(nodeTuple.authCache);
    client.execute(requestProducer, asyncResponseConsumer, context, new FutureCallback<HttpResponse>() {
        @Override
        public void completed(HttpResponse httpResponse) {
            try {
                RequestLogger.logResponse(logger, request, node.getHost(), httpResponse);
                int statusCode = httpResponse.getStatusLine().getStatusCode();
                Response response = new Response(request.getRequestLine(), node.getHost(), httpResponse);
                if (isSuccessfulResponse(statusCode) || ignoreErrorCodes.contains(response.getStatusLine().getStatusCode())) {
                    onResponse(node);
                    if (strictDeprecationMode && response.hasWarnings()) {
                        listener.onDefinitiveFailure(new ResponseException(response));
                    } else {
                        listener.onSuccess(response);
                    }
                } else {
                    ResponseException responseException = new ResponseException(response);
                    if (isRetryStatus(statusCode)) {
                        //mark host dead and retry against next one
                        onFailure(node);
                        retryIfPossible(responseException);
                    } else {
                        //mark host alive and don't retry, as the error should be a request problem
                        onResponse(node);
                        listener.onDefinitiveFailure(responseException);
                    }
                }
            } catch(Exception e) {
                listener.onDefinitiveFailure(e);
            }
        }

        @Override
        public void failed(Exception failure) {
            try {
                RequestLogger.logFailedRequest(logger, request, node, failure);
                onFailure(node);
                retryIfPossible(failure);
            } catch(Exception e) {
                listener.onDefinitiveFailure(e);
            }
        }

        private void retryIfPossible(Exception exception) {
            if (nodeTuple.nodes.hasNext()) {
                //in case we are retrying, check whether maxRetryTimeout has been reached
                long timeElapsedMillis = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime);
                long timeout = maxRetryTimeoutMillis - timeElapsedMillis;
                if (timeout <= 0) {
                    IOException retryTimeoutException = new IOException(
                            "request retries exceeded max retry timeout [" + maxRetryTimeoutMillis + "]");
                    listener.onDefinitiveFailure(retryTimeoutException);
                } else {
                    listener.trackFailure(exception);
                    request.reset();
                    performRequestAsync(startTime, nodeTuple, request, ignoreErrorCodes, httpAsyncResponseConsumerFactory, listener);
                }
            } else {
                listener.onDefinitiveFailure(exception);
            }
        }

        @Override
        public void cancelled() {
            listener.onDefinitiveFailure(new ExecutionException("request was cancelled", null));
        }
    });
}
```

可以看到，nodeTuple.nodes.next() 先从列表中取出下一个节点，接着构造并执行 Http 异步请求。而在请求响应回调函数中，RestClient 首先对请求响应码进行判断，如果请求响应码为成功（isSuccessfulResponse(statusCode)）或在用户指定的忽略列表里（ignoreErrorCodes.contains()），则表明请求正常，其他请求响应码表明结果异常。此时，首先判断是否是需要重试的响应码（isRetryStatus(statusCode)），如果不是，则返回错误信息给用户，如果是，则进行重试流程：retryIfPossible()，而 retryIfPossible() 逻辑也非常简单，除了重试超时时间判断部分，只是在方法里面又调用了一次 performRequestAsync()，再一次执行上面的流程。其中 isRetryStatus 的代码如下：
```java
private static boolean isRetryStatus(int statusCode) {
    switch(statusCode) {
        case 502:
        case 503:
        case 504:
            return true;
    }
    return false;
}
```

因此，由上述代码分析可知，在 Java REST Client 中，当发送用户请求异常时，只要服务端响应码不在用户指定的忽略响应码列表中且响应码在 isRetryStatus 范围内（502/503/504），并且未出现重试超时，RestClient 就会对请求进行重试，并一直重试至所有的可用节点都重试过为止。

## 4、Elasticsearch Clients 客户端请求负载均衡机制
在 Java API（TransportClient）中，es 客户端负载均衡策略类似于轮询（rr），其通过一个递增的序号并取模节点个数来实现。逐个节点发送请求：
```java
private int getNodeNumber() {
    int index = randomNodeGenerator.incrementAndGet();
    if (index < 0) {
        index = 0;
        randomNodeGenerator.set(0);
    }
    return index;
}

final DiscoveryNode getNode(int i) {
    return nodes.get((index + i) % nodes.size());
}
```

在 Java REST Client 中策略也类似，不过实现上使用了 Collections 的 rotate() 方法：
```java
    static Iterable<Node> selectNodes(NodeTuple<List<Node>> nodeTuple, Map<HttpHost, DeadHostState> blacklist,
                                      AtomicInteger lastNodeIndex, NodeSelector nodeSelector) throws IOException {
        ...
        if (false == livingNodes.isEmpty()) {
            List<Node> selectedLivingNodes = new ArrayList<>(livingNodes);
            nodeSelector.select(selectedLivingNodes);
            if (false == selectedLivingNodes.isEmpty()) {
                /*
                 * Rotate the list using a global counter as the distance so subsequent
                 * requests will try the nodes in a different order.
                 */
                Collections.rotate(selectedLivingNodes, lastNodeIndex.getAndIncrement());
                return selectedLivingNodes;
            }
        }
    }
```

## 5、Elasticsearch Clients 是否使用的是长连接？
由前述探活机制代码中可知，TransportClient 会在探活检测时提前建立节点连接并保持，因此使用的是长连接。

Java REST Client 使用的是默认 CloseableHttpAsyncClient，可看其创建时的 build() 代码：
```java
    public CloseableHttpAsyncClient build() {
        ...
        NHttpClientConnectionManager connManager = this.connManager;
        if (connManager == null) {
            ...
            final PoolingNHttpClientConnectionManager poolingmgr = new PoolingNHttpClientConnectionManager(
                    ioreactor,
                    RegistryBuilder.<SchemeIOSessionStrategy>create()
                        .register("http", NoopIOSessionStrategy.INSTANCE)
                        .register("https", sslStrategy)
                        .build());
            if (defaultConnectionConfig != null) {
                poolingmgr.setDefaultConnectionConfig(defaultConnectionConfig);
            }
            if (systemProperties) {
                String s = System.getProperty("http.keepAlive", "true");
                if ("true".equalsIgnoreCase(s)) {
                    s = System.getProperty("http.maxConnections", "5");
                    final int max = Integer.parseInt(s);
                    poolingmgr.setDefaultMaxPerRoute(max);
                    poolingmgr.setMaxTotal(2 * max);
                }
            } else {
            ...
            connManager = poolingmgr;
        }
        ConnectionReuseStrategy reuseStrategy = this.reuseStrategy;
        if (reuseStrategy == null) {
            if (systemProperties) {
                final String s = System.getProperty("http.keepAlive", "true");
                if ("true".equalsIgnoreCase(s)) {
                    reuseStrategy = DefaultConnectionReuseStrategy.INSTANCE;
                } else {
                    reuseStrategy = NoConnectionReuseStrategy.INSTANCE;
                }
            } else {
                reuseStrategy = DefaultConnectionReuseStrategy.INSTANCE;
            }
        }
        ConnectionKeepAliveStrategy keepAliveStrategy = this.keepAliveStrategy;
        if (keepAliveStrategy == null) {
            keepAliveStrategy = DefaultConnectionKeepAliveStrategy.INSTANCE;
        }
        ...
        return new InternalHttpAsyncClient(
            connManager,
            reuseStrategy,
            keepAliveStrategy,
            ...
    }
```

可见其默认用的连接管理器为 PoolingNHttpClientConnectionManager，并且默认开启了连接重用及 keepAlive，因此使用的也是长连接。

## 6、Elasticsearch Clients sniff 机制
默认情况下只有 Java API（TransportClient）有 sniff 机制，可查看上面 TransportClient 探活检测一节中代码描述。其作用可用于自动发现 data 类型节点，并同时往 data 节点发送客户端请求，以期提高并发性能（通常已区分 master/data/client 角色节点的集群不鼓励这么做）。Java REST Client 使用 sniff 需添加额外依赖，默认情况下不提供 sniff 功能，因此此处不再叙述。

