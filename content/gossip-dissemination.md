# Gossip 传播（Gossip Dissemination）

**原文**

https://martinfowler.com/articles/patterns-of-distributed-systems/gossip-dissemination.html

使用节点的随机选择进行信息传递，以确保信息可以到达集群中的所有节点，而不会淹没网络。

**2021.6.17**

## 问题

在拥有多个节点的集群中，每个节点都要向集群中的所有其它节点传递其所拥有的元数据信息，无需依赖于共享存储。在一个很大的集群里，如果所有的服务器要和所有其它的服务器通信，就会消耗大量的网络带宽。信息应该能够到达所有节点，即便有些网络连接遇到一些问题。

在大型的集群中，需要考虑下面一些东西：

* 对每台服务器产生的信息数量进行固定的限制
* 消息不应消耗大量的网络带宽。应该有一个上限，比如说几百 Kbs，确保不会因为集群中有过多的消息影响到应用的数据传输。
* 元数据的传播应该可以容忍网络和部分服务器的失效。即便有一些网络链接中断，或是有部分服务器失效，消息也能到达所有的服务器节点。

正如边栏中所讨论的，Gossip 式的通信满足了所有这些要求。

每个集群节点都把元数据存储为一个键值对列表，每个键值都到关联集群的一个节点，就像下面这样：

```java
class Gossip…

  Map<NodeId, NodeState> clusterMetadata = new HashMap<>();
class NodeState…

  Map<String, VersionedValue> values = new HashMap<>();
```

启动时，每个集群节点都会添加关于自己的元数据，这些元数据需要传播给其他节点。元数据的一个例子可以是节点监听的 IP 地址和端口，它负责的分区等等。Gossip 实例需要知晓至少一个其它节点的情况，以便开始进行 Gossip 通信。有一个集群节点需要众所周知，用于初始化 Gossip 实例，这个节点称为种子节点（a seed node），或者创始节点（introducer）。任何节点都可以充当创始节点。

```java
class Gossip…

  public Gossip(InetAddressAndPort listenAddress,
                List<InetAddressAndPort> seedNodes,
                String nodeId) throws IOException {
      this.listenAddress = listenAddress;
      //filter this node itself in case its part of the seed nodes
      this.seedNodes = removeSelfAddress(seedNodes);
      this.nodeId = new NodeId(nodeId);
      addLocalState(GossipKeys.ADDRESS, listenAddress.toString());

      this.socketServer = new NIOSocketListener(newGossipRequestConsumer(), listenAddress);
  }

  private void addLocalState(String key, String value) {
      NodeState nodeState = clusterMetadata.get(listenAddress);
      if (nodeState == null) {
          nodeState = new NodeState();
          clusterMetadata.put(nodeId, nodeState);
      }
      nodeState.add(key, new VersionedValue(value, incremenetVersion()));
  }
```

每个集群节点都会调度一个 job 用以定期将其拥有的元数据传输给其他节点。

```java
class Gossip…

  private ScheduledThreadPoolExecutor gossipExecutor = new ScheduledThreadPoolExecutor(1);
  private long gossipIntervalMs = 1000;
  private ScheduledFuture<?> taskFuture;
  public void start() {
      socketServer.start();
      taskFuture = gossipExecutor.scheduleAtFixedRate(()-> doGossip(),
                  gossipIntervalMs,
                  gossipIntervalMs,
                  TimeUnit.MILLISECONDS);
  }
```

调用调度任务时，它会从元数据集合的服务器列表中随机选取一小群节点。我们会定义一个小的常数，称为 Gossip 扇出，它会确定会选取多少节点称为 Gossip 的目标。如果什么都不知道，它会随机选取一个种子节点，然后发送其拥有的元数据集合给该节点。

```java
class Gossip…

  public void doGossip() {
      List<InetAddressAndPort> knownClusterNodes = liveNodes();
      if (knownClusterNodes.isEmpty()) {
          sendGossip(seedNodes, gossipFanout);
      } else {
          sendGossip(knownClusterNodes, gossipFanout);
      }
  }

  private List<InetAddressAndPort> liveNodes() {
      Set<InetAddressAndPort> nodes
              = clusterMetadata.values()
              .stream()
              .map(n -> InetAddressAndPort.parse(n.get(GossipKeys.ADDRESS).getValue()))
              .collect(Collectors.toSet());
      return removeSelfAddress(nodes);
  }

private void sendGossip(List<InetAddressAndPort> knownClusterNodes, int gossipFanout) {
    if (knownClusterNodes.isEmpty()) {
        return;
    }

    for (int i = 0; i < gossipFanout; i++) {
        InetAddressAndPort nodeAddress = pickRandomNode(knownClusterNodes);
        sendGossipTo(nodeAddress);
    }
}

private void sendGossipTo(InetAddressAndPort nodeAddress) {
    try {
        getLogger().info("Sending gossip state to " + nodeAddress);
        SocketClient<RequestOrResponse> socketClient = new SocketClient(nodeAddress);
        GossipStateMessage gossipStateMessage
                = new GossipStateMessage(this.clusterMetadata);
        RequestOrResponse request
                = createGossipStateRequest(gossipStateMessage);
        byte[] responseBytes = socketClient.blockingSend(request);
        GossipStateMessage responseState = deserialize(responseBytes);
        merge(responseState.getNodeStates());

    } catch (IOException e) {
        getLogger().error("IO error while sending gossip state to " + nodeAddress, e);
    }
}

private RequestOrResponse createGossipStateRequest(GossipStateMessage gossipStateMessage) {
    return new RequestOrResponse(RequestId.PushPullGossipState.getId(),
            JsonSerDes.serialize(gossipStateMessage), correlationId++);
}
```

接收 Gossip 消息的集群节点会检查其拥有的元数据，发现三件事。

* 传入消息中的值，且不再该节点状态集合中
* 该节点拥有，但不再传入的 Gossip 消息中
* 节点拥有传入消息的值，这时会选择版本更高的值

稍后，它会将缺失的值添加到自己的状态集合中。传入消息中若有任何值缺失，就会在应答中返回这些值。

发送 Gossip 消息的集群节点会将从 Gossip 应答中得到值添加到自己的状态中。

```java
class Gossip…

  private void handleGossipRequest(org.distrib.patterns.common.Message<RequestOrResponse> request) {
      GossipStateMessage gossipStateMessage = deserialize(request.getRequest());
      Map<NodeId, NodeState> gossipedState = gossipStateMessage.getNodeStates();
      getLogger().info("Merging state from " + request.getClientSocket());
      merge(gossipedState);

      Map<NodeId, NodeState> diff = delta(this.clusterMetadata, gossipedState);
      GossipStateMessage diffResponse = new GossipStateMessage(diff);
      getLogger().info("Sending diff response " + diff);
      request.getClientSocket().write(new RequestOrResponse(RequestId.PushPullGossipState.getId(),
                      JsonSerDes.serialize(diffResponse),
                      request.getRequest().getCorrelationId()));
  }
public Map<NodeId, NodeState> delta(Map<NodeId, NodeState> fromMap, Map<NodeId, NodeState> toMap) {
    Map<NodeId, NodeState> delta = new HashMap<>();
    for (NodeId key : fromMap.keySet()) {
        if (!toMap.containsKey(key)) {
            delta.put(key, fromMap.get(key));
            continue;
        }
        NodeState fromStates = fromMap.get(key);
        NodeState toStates = toMap.get(key);
        NodeState diffStates = fromStates.diff(toStates);
        if (!diffStates.isEmpty()) {
            delta.put(key, diffStates);
        }
    }
    return delta;
}
public void merge(Map<NodeId, NodeState> otherState) {
    Map<NodeId, NodeState> diff = delta(otherState, this.clusterMetadata);
    for (NodeId diffKey : diff.keySet()) {
        if(!this.clusterMetadata.containsKey(diffKey)) {
            this.clusterMetadata.put(diffKey, diff.get(diffKey));
        } else {
            NodeState stateMap = this.clusterMetadata.get(diffKey);
            stateMap.putAll(diff.get(diffKey));
        }
    }
}
```

每隔一秒，这个过程就会在集群的每个节点上发生一次，每次都会选择不同的节点进行状态交换。