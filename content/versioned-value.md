# 有版本的值（Versioned Value）

**原文**

https://martinfowler.com/articles/patterns-of-distributed-systems/versioned-value.html

Store every update to a value with a new version, to allow reading historical values.

将值的每次更新连同新版本一同存储起来，以允许读取历史值。

**2021.6.22**

## 问题

在分布式系统中，节点需要知道对于某个键值而言，哪个值是最新的。有时，它们知道过去的值，这样，它们就可以对一个值的变化做出恰当的反应了。


## 解决方案

在每个值里面都存储一个版本号。版本在每次更新时递增。这样，每次更新就都可以转换为一次新的写入，而无需阻塞读取。客户端可以读取特定版本号的对应的历史值。

考虑一个复制的键值存储的简单例子。集群的领导者处理所有对键值存储的写入。它将写入请求保存在[预写日志](write-ahead-log.md)中。预写日志使用[领导者和追随者](leader-and-followers.md)进行复制。在到达[高水位标记](high-water-mark.md)时，领导者将预写日志的条目应用到键值存储中。这是一种标准的复制方法，称之为[状态机复制（state-machine-replication）](https://en.wikipedia.org/wiki/State_machine_replication)。大多数数据系统，其背后如果像 Raft 这样的共识算法支撑，都是这样实现的。在这种情况下，键值存储中会保存一个整数的版本计数器。每次根据预写日志应用键值与值的写命令时，这个计数器都要递增。然后，它会根据递增之后的版本计数器，构建一个新的键值。这样一来，不存在值的更新，但每次写入请求都会向后面的存储中附加一个新的值。


```java
class ReplicatedKVStore…

  int version = 0;
  MVCCStore mvccStore = new MVCCStore();

  @Override
  public CompletableFuture<Response> put(String key, String value) {
      return server.propose(new SetValueCommand(key, value));
  }

  private Response applySetValueCommand(SetValueCommand setValueCommand) {
      getLogger().info("Setting key value " + setValueCommand);
      version = version + 1;
      mvccStore.put(new VersionedKey(setValueCommand.getKey(), version), setValueCommand.getValue());
      Response response = Response.success(version);
      return response;
  }
```

## 有版本键值的排序

能够快速定位到最佳匹配的版本，这是一个重要的实现考量，所以，有版本的价值常按这种方式进行组织：使用版本号当做键值的后缀，形成一个自然排序。这样就可以保持一个与底层数据结构相适应的顺序。比如说，一个键值有两个版本，key1 和 key2，key1 就会排在 key2 前面。


要存储有版本的键值与值，可以使用某种数据结构，比如，跳表，这样可以快速定位到最近的匹配版本上。使用 Java，可以像下面这样构建 MVCC 存储：

```java
class MVCCStore…

  public class MVCCStore {
      NavigableMap<VersionedKey, String> kv = new ConcurrentSkipListMap<>();
  
      public void put(VersionedKey key, String value) {
          kv.put(key, value);
  
      }

```

为了使用 NavigableMap，有版本的键值可以像下面这样实现。它会实现一个比较器，允许键值的自然排序。

```java
class VersionedKey…

  public class VersionedKey implements Comparable<VersionedKey> {
      private String key;
      private int version;
  
      public VersionedKey(String key, int version) {
          this.key = key;
          this.version = version;
      }
  
      public String getKey() {
          return key;
      }
  
      public int getVersion() {
          return version;
      }
  
      @Override
      public int compareTo(VersionedKey other) {
          int keyCompare = this.key.compareTo(other.key);
          if (keyCompare != 0) {
              return keyCompare;
          }
          return Integer.compare(this.version, other.version);
      }
  }

```

这个实现允许通过 NavigableMap 的 API 获取特定版本的值。

```java
class MVCCStore…

  public Optional<String> get(final String key, final int readAt) {
      Map.Entry<VersionedKey, String> entry = kv.floorEntry(new VersionedKey(key, readAt));
      return (entry == null)? Optional.empty(): Optional.of(entry.getValue());
  }

```

看一个例子，一个键值有四个版本，存储的版本号分别是 1、2、3 和 5。根据客户端所使用版本去读取值，返回的是最接近匹配版本的键值。

![读取特定版本](../image/versioned-key-read.png)
<center>图1：读取特定版本</center>


存储有特定键值和值的版本就会返回给客户端。客户端使用这个版本读取值。整体工作情况如下所示：

versioned-value-logical-clock-put

![Put 请求处理](../image/versioned-value-logical-clock-put.svg)
<center>图2：Put 请求处理</center>

![Put 请求处理](../image/versioned-value-logical-clock-get.svg)
<center>图3：读取特定版本</center>

# 读取多个版本

有时，客户端需要获取从某个给定版本号开始的所有版本。比如，在[状态监控](content/state-watch.md)中，客户端就要获取从指定版本开始的所有事件。

集群节点可以多存一个索引结构，以便将一个键值所有的版本都存起来。

```java
class IndexedMVCCStore…

  public class IndexedMVCCStore {
      NavigableMap<String, List<Integer>> keyVersionIndex = new TreeMap<>();
      NavigableMap<VersionedKey, String> kv = new TreeMap<>();

      ReadWriteLock rwLock = new ReentrantReadWriteLock();
      int version = 0;

      public int put(String key, String value) {
          rwLock.writeLock().lock();
          try {
              version = version + 1;
              kv.put(new VersionedKey(key, version), value);

              updateVersionIndex(key, version);

              return version;
          } finally {
              rwLock.writeLock().unlock();
          }
      }

      private void updateVersionIndex(String key, int newVersion) {
          List<Integer> versions = getVersions(key);
          versions.add(newVersion);
          keyVersionIndex.put(key, versions);
      }

      private List<Integer> getVersions(String key) {
          List<Integer> versions = keyVersionIndex.get(key);
          if (versions == null) {
              versions = new ArrayList<>();
              keyVersionIndex.put(key, versions);
          }
          return versions;
      }
```

这样，就可以提供一个客户端 API，读取从指定版本开始或者一个版本范围内的所有值。

```java
class IndexedMVCCStore…
  public List<String> getRange(String key, final int fromRevision, int toRevision) {
      rwLock.readLock().lock();
      try {
          List<Integer> versions = keyVersionIndex.get(key);
          Integer maxRevisionForKey = versions.stream().max(Integer::compareTo).get();
          Integer revisionToRead = maxRevisionForKey > toRevision ? toRevision : maxRevisionForKey;
          SortedMap<VersionedKey, String> versionMap = kv.subMap(new VersionedKey(key, revisionToRead), new VersionedKey(key, toRevision));
          getLogger().info("Available version keys " + versionMap + ". Reading@" + fromRevision + ":" + toRevision);
          return new ArrayList<>(versionMap.values());

      } finally {
          rwLock.readLock().unlock();
      }
  }

```

有一点必须多加小心，根据某个索引进行相应的更新和读取时，需要使用恰当的锁。

There is an alternate implementation possible to save a list of all the versioned values with the key, as used in Gossip Dissemination to avoid unnecessary state exchange.

有一种替代的实现方案，就是将所有值的列表和键值存在一起，就像在Gossip 传播（Gossip Dissemination）中所用的一样，以[规避不必要的状态交换](https://martinfowler.com/articles/patterns-of-distributed-systems/gossip-dissemination.html#AvoidingUnnecessaryStateExchange)

