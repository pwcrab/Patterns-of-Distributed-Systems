# Lamport 时钟（Lamport Clock）

**原文**

https://martinfowler.com/articles/patterns-of-distributed-systems/lamport-clock.html

使用逻辑时间戳作为一个值的版本，以便支持跨服务器的值排序。

**2021.6.23**

## 问题

当值要在多个服务器上进行存储时，需要有一种方式知道一个值要在另外一个值之前存储。在这种情况下，不能使用系统时间戳，因为时钟不是单调的，两个服务器的时钟时间不应该进行比较。

表示一天中时间的系统时间戳，一般来说是通过晶体振荡器建造的时钟机械测量的。这种机制有一个已知问题，根据晶体震荡的快慢，它可能会偏离一天实际的时间。为了解决这个问题，计算机通常会使用像 NTP 这样的服务，将计算机时钟与互联网上众所周知的时间源进行同步。正因为如此，在一个给定的服务器上连续读取两次系统时间，可能会出现时间倒退的现象。

由于服务器之间的时钟漂移没有上限，比较两个不同的服务器的时间戳是不可能的。

## 解决方案

Lamport 时钟维护着一个单独的数字表示时间戳，如下所示：

```java
class LamportClock…

  class LamportClock {
      int latestTime;

      public LamportClock(int timestamp) {
          latestTime = timestamp;
      }
```

每个集群节点都维护着一个 Lamport 时钟的实例。

```java
class Server…

  MVCCStore mvccStore;
  LamportClock clock;

  public Server(MVCCStore mvccStore) {
      this.clock = new LamportClock(1);
      this.mvccStore = mvccStore;
  }
```

服务器每当进行任何写操作时，它都应该使用`tick()`方法让 Lamport 时钟前进。

```java
class LamportClock…

  public int tick(int requestTime) {
      latestTime = Integer.max(latestTime, requestTime);
      latestTime++;
      return latestTime;
  }
```

如此一来，服务器可以确保写操作的顺序是在这个请求之后，以及客户端发起请求时服务器端已经执行的任何其他动作之后。服务器会返回一个时间戳，用于将值写回给客户端。稍后，请求的客户端会使用这个时间戳向其它的服务器发起进一步的写操作。如此一来，请求的因果链就得到了维持。