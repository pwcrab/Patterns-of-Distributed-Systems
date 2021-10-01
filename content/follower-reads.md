# 追随者读取（Follower Reads）

**原文**

https://martinfowler.com/articles/patterns-of-distributed-systems/follower-reads.html

为来自追随者的读取请求提供服务，获取更好的吞吐和更低的延迟。

**2021.7.1**

## 问题

使用领导者和追随者模式时，如果有太多请求发给领导者，它可能会出现过载。此外，在多数据中心的情况下，客户端如果在远程的数据中心，向领导者发送的请求可能会有额外的延迟。

## 解决方案

当写请求要到领导者那去维持一致性，只读请求就会转到最近的追随者。当客户端大多都是只读的，这种做法就特别有用。

值得注意的是，从追随者那里读取的客户端得到可能是旧值。领导者和追随者之间总会存在一个复制滞后，即使是在像 [Raft](https://raft.github.io/) 这样实现共识算法的系统中。这是因为即使领导者知道哪些值提交过了，它也需要另外一个消息将这个信息传达给跟随者。因此，从追随者服务器上读取信息只能用于“可以容忍稍旧的值”的情况。

![从最近的追随者上读取](../image/follower-reads.png)
<center>图 1：从最近的追随者上读取</center>

### 找到最近的节点

集群节点要额外维护其位置的元数据。

```java
class ReplicaDescriptor…

  public class ReplicaDescriptor {
      public ReplicaDescriptor(InetAddressAndPort address, String region) {
          this.address = address;
          this.region = region;
      }
      InetAddressAndPort address;
      String region;

      public InetAddressAndPort getAddress() {
          return address;
      }

      public String getRegion() {
          return region;
      }
  }
```

然后，集群客户端可以根据自己的区域选取本地的副本。

```java
class ClusterClient…

  public List<String> get(String key) {
      List<ReplicaDescriptor> allReplicas = allFollowerReplicas(key);
      ReplicaDescriptor nearestFollower = findNearestFollowerBasedOnLocality(allReplicas, clientRegion);
      GetValueResponse getValueResponse = sendGetRequest(nearestFollower.getAddress(), new GetValueRequest(key));
      return getValueResponse.getValue();
  }

  ReplicaDescriptor findNearestFollowerBasedOnLocality(List<ReplicaDescriptor> followers, String clientRegion) {
      List<ReplicaDescriptor> sameRegionFollowers = matchLocality(followers, clientRegion);
      List<ReplicaDescriptor> finalList = sameRegionFollowers.isEmpty()?followers:sameRegionFollowers;
      return finalList.get(0);
  }

  private List<ReplicaDescriptor> matchLocality(List<ReplicaDescriptor> followers, String clientRegion) {
      return followers.stream().filter(rd -> clientRegion.equals(rd.region)).collect(Collectors.toList());
  }
```

例如，如果有两个追随者副本，一个在美国西部（us-west），另一个在美国东部（us-east）。美国东部的客户端就会连接到美国东部的副本上。

```java
class CausalKVStoreTest…

  @Test
  public void getFollowersInSameRegion() {
      List<ReplicaDescriptor> followers = createReplicas("us-west", "us-east");
      ReplicaDescriptor nearestFollower = new ClusterClient(followers, "us-east").findNearestFollower(followers);
      assertEquals(nearestFollower.getRegion(), "us-east");

  }
```

集群客户端或协调的集群节点也会追踪其同集群节点之间可观察的延迟。它可以发送周期性的心跳来获取延迟，并根据它选出延迟最小的追随者。为了做一个更公平的选择，像 [mongodb](https://www.mongodb.com/) 或 [cockroachdb](https://www.cockroachlabs.com/docs/stable/) 这样的产品会把延迟当做滑动平均来计算。集群节点一般会同其它集群节点之间维护一个[单一 Socket 通道（Single Socket Channel）](single-socket-channel.md)进行通信。单一 Socket 通道（Single Socket Channel）会使用心跳（HeartBeat）进行连接保活。因此，获取延迟和计算滑动移动平均就可以很容易地实现。

```java
class WeightedAverage…

  public class WeightedAverage {
      long averageLatencyMs = 0;
      public void update(long heartbeatRequestLatency) {
          //Example implementation of weighted average as used in Mongodb
          //The running, weighted average round trip time for heartbeat messages to the target node.
          // Weighted 80% to the old round trip time, and 20% to the new round trip time.
          averageLatencyMs = averageLatencyMs == 0
                  ? heartbeatRequestLatency
                  : (averageLatencyMs * 4 + heartbeatRequestLatency) / 5;
      }

      public long getAverageLatency() {
          return averageLatencyMs;
      }
  }
class ClusterClient…

  private Map<InetAddressAndPort, WeightedAverage> latencyMap = new HashMap<>();
  private void sendHeartbeat(InetAddressAndPort clusterNodeAddress) {
      try {
          long startTimeNanos = System.nanoTime();
          sendHeartbeatRequest(clusterNodeAddress);
          long endTimeNanos = System.nanoTime();

          WeightedAverage heartbeatStats = latencyMap.get(clusterNodeAddress);
          if (heartbeatStats == null) {
              heartbeatStats = new WeightedAverage();
              latencyMap.put(clusterNodeAddress, new WeightedAverage());
          }
          heartbeatStats.update(endTimeNanos - startTimeNanos);

      } catch (NetworkException e) {
          logger.error(e);
      }
  }
```

This latency information can then be used to pick up the follower with the least network latency.

```java
class ClusterClient…

  ReplicaDescriptor findNearestFollower(List<ReplicaDescriptor> allFollowers) {
      List<ReplicaDescriptor> sameRegionFollowers = matchLocality(allFollowers, clientRegion);
      List<ReplicaDescriptor> finalList
              = sameRegionFollowers.isEmpty() ? allFollowers
                                                :sameRegionFollowers;
      return finalList.stream().sorted((r1, r2) -> {
          if (!latenciesAvailableFor(r1, r2)) {
              return 0;
          }
          return Long.compare(latencyMap.get(r1).getAverageLatency(),
                              latencyMap.get(r2).getAverageLatency());

      }).findFirst().get();
  }

  private boolean latenciesAvailableFor(ReplicaDescriptor r1, ReplicaDescriptor r2) {
      return latencyMap.containsKey(r1) && latencyMap.containsKey(r2);
  }
```

这样，就可以利用延迟信息选取网络延迟最小的追随者。

```java
class ClusterClient…

  ReplicaDescriptor findNearestFollower(List<ReplicaDescriptor> allFollowers) {
      List<ReplicaDescriptor> sameRegionFollowers = matchLocality(allFollowers, clientRegion);
      List<ReplicaDescriptor> finalList
              = sameRegionFollowers.isEmpty() ? allFollowers
                                                :sameRegionFollowers;
      return finalList.stream().sorted((r1, r2) -> {
          if (!latenciesAvailableFor(r1, r2)) {
              return 0;
          }
          return Long.compare(latencyMap.get(r1).getAverageLatency(),
                              latencyMap.get(r2).getAverageLatency());

      }).findFirst().get();
  }

  private boolean latenciesAvailableFor(ReplicaDescriptor r1, ReplicaDescriptor r2) {
      return latencyMap.containsKey(r1) && latencyMap.containsKey(r2);
  }
```

### 断连或缓慢的追随者

追随者可能会与领导者之间失去联系，停止获得更新。在某些情况下，追随者可能会受到慢速磁盘的影响，阻碍整个的复制过程，这会导致追随者滞后于领导者。追随者追踪到其是否有一段时间没有收到领导者的消息，在这种情况下，可以停止对用户请求进行服务。

比如，像 [mongodb](https://www.mongodb.com/) 这样的产品会选择带有[最大可接受滞后时间（maximum allowed lag time）](https://docs.mongodb.com/manual/core/read-preference-staleness/#std-label-replica-set-read-preference-max-staleness)的副本。如果副本滞后于领导者超过了这个最大时间，就不会选择它继续对用户请求提供服务。在 [kafka](https://kafka.apache.org/) 中，如果追随者检测到消费者请求的偏移量过大，它就会给出一个 OFFSET_OUT_OF_RANGE 的错误。我们就预期消费者会与领导者进行通信。